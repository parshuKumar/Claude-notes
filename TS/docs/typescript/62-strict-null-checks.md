# 62 — Strict Null Checks

## What is this?

`strictNullChecks` is a `tsconfig.json` compiler flag. When it is **off**, `null` and `undefined` are members of *every* type — a `string` can be `null`, a `User` can be `undefined`, and TypeScript says nothing. When it is **on**, `null` and `undefined` get their own distinct types and are only assignable to types that explicitly include them.

```ts
// ── strictNullChecks: false ─────────────────────────────────────────────────
let email: string = null;       // allowed — null is assignable to string
let userId: number = undefined; // allowed — undefined is assignable to number

// ── strictNullChecks: true ──────────────────────────────────────────────────
let email: string = null;              // ❌ Type 'null' is not assignable to type 'string'
let userId: number = undefined;        // ❌ Type 'undefined' is not assignable to type 'number'

let maybeEmail: string | null = null;       // ✅ explicitly nullable
let maybeUserId: number | undefined;        // ✅ explicitly optional (implicitly undefined)
```

That single flag is the difference between TypeScript being a nicer JSDoc and TypeScript being a tool that eliminates an entire class of production incidents. It is turned on by `"strict": true`, and it is the single most valuable flag in the whole compiler.

Around it sits a family of related flags and syntax:

- `strictPropertyInitialization` — class fields must actually be assigned
- `noUncheckedIndexedAccess` — `array[i]` and `record[key]` might not be there
- `!` — the non-null assertion operator (the escape hatch that lies)
- `!:` — definite assignment assertion on declarations
- `?.` and `??` — the runtime operators TypeScript understands deeply

## Why does it matter?

`TypeError: Cannot read properties of undefined (reading 'email')` is the single most common runtime crash in Node backends. Every one of them is a place where a value was `null` or `undefined` and the code assumed it was not.

Backend code is *drowning* in optionality:

- `db.findUserById(userId)` returns a user **or nothing** — the row may not exist
- `req.headers.authorization` may be absent — clients forget headers
- `req.body.email` may be missing — clients send garbage
- `process.env.DATABASE_URL` is `string | undefined` — nobody set the variable in staging
- `arr.find(u => u.id === userId)` returns `User | undefined` — always
- `JSON.parse(payload).data?.items?.[0]` — three chances to explode
- A nullable database column maps to `string | null` — that is what `NULL` means
- `new Map<string, Session>().get(sessionId)` returns `Session | undefined`

With `strictNullChecks: false`, TypeScript happily lets you call `.email` on all of them and the crash happens at 3am in production. With it on, the compiler refuses to build until you have decided, at *each* site, what happens when the value is absent. It converts a runtime incident into a compile error, which is the entire point of using TypeScript at all.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript — every access is a gamble ─────────────────────────────────────

async function getUserProfile(userId) {
  const user = await db.findUserById(userId);

  // Does user exist? Who knows. The function signature tells you nothing.
  return {
    id:        user.id,
    email:     user.email,
    // Nested nullability — profile may be null, avatar may be null:
    avatarUrl: user.profile.avatar.url,
    // A nullable column — this is `null` for half your rows:
    fullName:  user.firstName + " " + user.lastName,
  };
}

// Runtime, in production, on the 4th request:
//   TypeError: Cannot read properties of null (reading 'id')
//   at getUserProfile (/app/dist/services/user.js:12:15)

// So JS devs learn to write paranoid code EVERYWHERE, because there is no way
// to know which values are safe:
function getUserProfile(userId) {
  const user = db.findUserById(userId);
  if (!user) return null;
  if (!user.profile) return { id: user.id, email: user.email, avatarUrl: null };
  if (!user.profile.avatar) return { id: user.id, email: user.email, avatarUrl: null };
  // ...and you still forgot one, somewhere else in the file.
}
```

The JS problem is not that values can be missing — that is reality. The problem is that **nothing distinguishes a value that can be missing from one that cannot**, so you either check everything (noise, and you still miss cases) or check nothing (crashes).

```ts
// ── TypeScript with strictNullChecks — the type IS the documentation ──────────

interface User {
  id:        number;
  email:     string;             // never null — guaranteed by the type
  firstName: string;
  lastName:  string | null;      // nullable column in the DB — the type says so
  profile:   UserProfile | null; // 1-to-1 relation that may not exist
}

interface UserProfile {
  bio:       string | null;
  avatarUrl: string | null;
}

// The return type says "may not find one" — callers CANNOT ignore this:
async function findUserById(userId: number): Promise<User | null> {
  const row = await db.query("SELECT * FROM users WHERE id = $1", [userId]);
  return row ?? null;
}

async function getUserProfile(userId: number): Promise<ProfileDto | null> {
  const user = await findUserById(userId);

  if (!user) return null;   // ← the compiler FORCED you to write this line

  // From here down, `user` is `User` — not `User | null`. Zero doubt.
  return {
    id:        user.id,
    email:     user.email,                        // ✅ string, always
    avatarUrl: user.profile?.avatarUrl ?? null,   // ✅ handles both nulls in one line
    fullName:  user.lastName
      ? `${user.firstName} ${user.lastName}`      // ✅ narrowed to string
      : user.firstName,
  };
}
```

The revelation: you stop writing defensive checks *everywhere* and start writing them *exactly where the type says a value can be absent*. The compiler tracks it for you. Code that never checks is now **provably** code that never needed to check.

---

## Syntax

```ts
// ── Nullable and optional types ─────────────────────────────────────────────
let sessionId:  string | null;       // may hold null — must be assigned before use
let authToken:  string | undefined;  // may hold undefined
let requestId?: string;              // (in an interface) optional → string | undefined

interface RequestContext {
  requestId: string;                 // required, never undefined
  userId?:   number;                 // optional → number | undefined
  traceId:   string | null;          // required key, value may be null
}

// ── Narrowing: the ONLY safe way to get from `T | null` to `T` ──────────────
function greet(user: User | null): string {
  if (user === null) return "Hello, guest";  // narrow by explicit comparison
  return `Hello, ${user.email}`;             // user: User
}

function greet2(user: User | null): string {
  if (!user) return "Hello, guest";          // narrow by truthiness
  return `Hello, ${user.email}`;              // user: User
}

// ── Optional chaining ?. — short-circuits to undefined ──────────────────────
const avatar = user?.profile?.avatarUrl;     // string | null | undefined
const first  = users?.[0];                   // element | undefined
const result = callback?.(requestBody);      // return type | undefined

// ── Nullish coalescing ?? — falls back ONLY on null/undefined ───────────────
const port = envPort ?? 3000;                // 0 and "" are kept; null/undefined are not

// ── Non-null assertion ! — "trust me, it's there" (unchecked!) ──────────────
const definitelyUser = findUser(userId)!;    // strips null | undefined — NO runtime check

// ── Definite assignment assertion !: — "I assign this later" ────────────────
let connection!: DbConnection;               // no initializer, no error

class UserService {
  private repo!: UserRepository;             // assigned in an init() method
}
```

---

## How it works — concept by concept

### Concept 1 — What actually changes when the flag flips

With `strictNullChecks: false`, `null` and `undefined` are in the **domain of every type**. Assignability is a lie:

```ts
// strictNullChecks: false
function chargeCard(amountCents: number, cardToken: string): void { /* ... */ }

chargeCard(null, undefined);   // ✅ compiles. Charges null cents. 🎉

const users: User[] = null;
users.length;                  // ✅ compiles. 💥 at runtime.

// Worse: the compiler cannot even TELL you a value might be missing:
const user = users.find(u => u.id === 1);
//    ^? User   ← a lie. find() returns undefined when nothing matches.
user.email;                    // ✅ compiles. 💥 at runtime.
```

With `strictNullChecks: true`, `null` and `undefined` become their own types with almost no assignability:

```ts
// strictNullChecks: true
const user = users.find(u => u.id === userId);
//    ^? User | undefined   ← the truth, straight from lib.es5.d.ts

user.email;      // ❌ 'user' is possibly 'undefined'.
user?.email;     // ✅ string | undefined
if (user) user.email;  // ✅ string
```

The `undefined` in `Array.prototype.find`'s signature was *always there* in the lib definitions. `strictNullChecks: false` was silently deleting it from every type in the program.

### Concept 2 — `undefined` vs `null`: the convention that matters

TypeScript treats them as two distinct types, and they are **not** interchangeable:

```ts
let a: string | undefined = null;   // ❌ Type 'null' is not assignable
let b: string | null = undefined;   // ❌ Type 'undefined' is not assignable
```

So you must pick conventions. The one that scales in Node backends:

```ts
// ── undefined = "this key/value was never provided" ─────────────────────────
//    Sources: optional properties, missing function args, missing env vars,
//             Map.get() misses, array out-of-bounds, Array.find() misses.

interface CreateUserRequest {
  email:     string;
  password:  string;
  referrer?: string;              // undefined — the client did not send it
}

// ── null = "this value explicitly exists and is empty" ──────────────────────
//    Sources: SQL NULL columns, JSON payloads (JSON has null, not undefined),
//             "cleared" values, deliberate absence.

interface UserRow {
  id:            number;
  email:         string;
  deleted_at:    Date | null;     // SQL NULL — the row is not deleted
  last_login_at: Date | null;     // SQL NULL — never logged in
}
```

The practical consequence — `JSON.stringify` drops `undefined` keys but keeps `null`:

```ts
JSON.stringify({ userId: 1, referrer: undefined });  // '{"userId":1}'         ← key vanishes
JSON.stringify({ userId: 1, referrer: null });       // '{"userId":1,"referrer":null}'
```

So for an API contract where "the field is present and explicitly empty" is meaningful (a PATCH that clears a field), you **must** use `null`. Using `undefined` makes "clear this field" indistinguishable from "don't touch this field":

```ts
interface UpdateUserPatch {
  email?:     string;             // absent → leave unchanged
  bio?:       string | null;      // absent → unchanged; null → clear it; string → set it
  avatarUrl?: string | null;
}

function applyPatch(user: User, patch: UpdateUserPatch): User {
  return {
    ...user,
    // `in` narrows presence of the KEY, not truthiness of the value —
    // this is the only correct way to distinguish "absent" from "null":
    email:     "email"     in patch && patch.email     !== undefined ? patch.email     : user.email,
    bio:       "bio"       in patch ? patch.bio       ?? null : user.bio,
    avatarUrl: "avatarUrl" in patch ? patch.avatarUrl ?? null : user.avatarUrl,
  };
}
```

### Concept 3 — Optional chaining `?.` and what TypeScript infers

`?.` short-circuits the **entire chain** to `undefined` the moment any link is `null` or `undefined`:

```ts
interface Organization {
  id:      number;
  owner:   User | null;
  billing: BillingAccount | null;
}

interface BillingAccount {
  stripeCustomerId: string;
  defaultCard:      Card | null;
}

declare const org: Organization;

const cardLast4 = org.billing?.defaultCard?.last4;
//    ^? string | undefined
// Note: NOT `string | null | undefined` for the tail — the `null`s were consumed
// by the ?. operators and converted to `undefined`.

// Optional call — only calls if the function exists:
declare const onUserCreated: ((userId: number) => void) | undefined;
onUserCreated?.(userId);        // safe; expression type is `void | undefined`

// Optional index access:
declare const auditLog: string[] | undefined;
const firstEntry = auditLog?.[0];
//    ^? string | undefined

// ⚠️ ?. protects the ACCESS, not the whole expression:
const length = org.billing?.stripeCustomerId.length;
//    ^? number | undefined
// `stripeCustomerId` is non-nullable so no ?. needed there — the whole chain
// short-circuits at `billing?.` and the rest is skipped entirely.

// ⚠️ ?. does NOT make an assignment target safe:
// org.billing?.stripeCustomerId = "cus_1";   // ❌ not a valid assignment target
```

The key insight most tutorials skip: `a?.b.c` is not `a?.b?.c`. If `a` exists but `b` is `null`, the first form still crashes. `?.` short-circuits only on the link it is attached to.

### Concept 4 — Nullish coalescing `??` vs `||`

`||` falls back on any **falsy** value: `0`, `""`, `NaN`, `false`, `null`, `undefined`.
`??` falls back on **only** `null` and `undefined`.

```ts
interface RateLimitConfig {
  maxRequests: number;
  windowMs:    number;
  enabled:     boolean;
  keyPrefix:   string;
}

function buildConfig(overrides: Partial<RateLimitConfig>): RateLimitConfig {
  return {
    // ❌ WRONG — `|| 100` means you can never configure 0 requests:
    // maxRequests: overrides.maxRequests || 100,

    // ❌ WRONG — `|| true` means `enabled: false` is silently ignored:
    // enabled: overrides.enabled || true,

    // ❌ WRONG — `|| "rl:"` means an empty prefix is impossible:
    // keyPrefix: overrides.keyPrefix || "rl:",

    // ✅ RIGHT — only null/undefined trigger the fallback:
    maxRequests: overrides.maxRequests ?? 100,
    windowMs:    overrides.windowMs    ?? 60_000,
    enabled:     overrides.enabled     ?? true,
    keyPrefix:   overrides.keyPrefix   ?? "rl:",
  };
}

buildConfig({ enabled: false, maxRequests: 0 });
// with ?? → { maxRequests: 0, windowMs: 60000, enabled: false, keyPrefix: "rl:" }  ✅
// with || → { maxRequests: 100, windowMs: 60000, enabled: true, keyPrefix: "rl:" } ❌ silently wrong
```

TypeScript narrows the result type of `??` correctly:

```ts
declare const timeoutMs: number | undefined;
const timeout = timeoutMs ?? 5_000;
//    ^? number   ← undefined removed from the union

declare const overrideHost: string | null | undefined;
const host = overrideHost ?? process.env.HOST ?? "localhost";
//    ^? string   ← both null and undefined removed by the chain
```

`??` cannot be mixed with `||` or `&&` without parentheses — that is a syntax error, deliberately, because the precedence would be ambiguous:

```ts
// const x = a || b ?? c;      // ❌ SyntaxError
const x = (a || b) ?? c;       // ✅ explicit
```

There is also `??=` (logical nullish assignment):

```ts
const cache: Record<string, Session | undefined> = {};
function getSession(sessionId: string): Session {
  cache[sessionId] ??= createSession(sessionId);  // assign only if null/undefined
  return cache[sessionId]!;
}
```

### Concept 5 — The non-null assertion `!` and why it lies

`value!` tells the compiler "remove `null` and `undefined` from this type". It emits **zero runtime code**. It is a pure type-level override — you are overruling the compiler with your word.

```ts
declare function findUser(userId: number): User | undefined;

const user = findUser(userId)!;    // type: User
console.log(user.email);           // compiles fine
// If findUser returned undefined → 💥 TypeError at runtime.
// The `!` did NOT check anything. It compiled to: findUser(userId)
```

`!` is not "assert this is not null". It is "**stop telling me** this might be null". Those are opposite in safety.

Where `!` is genuinely defensible:

```ts
// ✅ 1. You just proved it, but the compiler cannot see across the call boundary:
const sessionMap = new Map<string, Session>();
if (sessionMap.has(sessionId)) {
  const session = sessionMap.get(sessionId)!;  // has() guarantees get() succeeds
  // (TS has no way to link has() and get() — this is a known limitation)
}

// ✅ 2. Test setup where a crash IS the desired failure mode:
const created = await userRepo.findByEmail("test@example.com");
expect(created!.id).toBeGreaterThan(0);
```

Where `!` is a bug waiting to happen — and what to write instead:

```ts
// ❌ Asserting on external input:
const authToken = req.headers.authorization!.split(" ")[1];
// 💥 whenever a client omits the header. Which is constantly.

// ✅ Handle it — it is three extra lines and it never pages you:
const authHeader = req.headers.authorization;
if (!authHeader) throw new UnauthorizedError("Missing Authorization header");
const [scheme, authToken] = authHeader.split(" ");
if (scheme !== "Bearer" || !authToken) throw new UnauthorizedError("Malformed Authorization header");

// ❌ Asserting on env vars:
const dbUrl = process.env.DATABASE_URL!;
// 💥 in staging, at connection time, with an unhelpful error.

// ✅ Fail loudly at boot, with a message that names the variable:
function requireEnv(name: string): string {
  const value = process.env[name];
  if (value === undefined || value === "") {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}
const dbUrl = requireEnv("DATABASE_URL");   // string — and a real runtime guarantee
```

Note `requireEnv` gives you the *same ergonomics* as `!` (a `string`, no narrowing at the call site) with an actual runtime check behind it. That is almost always the right trade.

You can ban `!` project-wide with the ESLint rule `@typescript-eslint/no-non-null-assertion`.

### Concept 6 — Definite assignment `!:` and `strictPropertyInitialization`

`strictPropertyInitialization` (also enabled by `strict: true`, and requires `strictNullChecks`) says: every class property whose type does not include `undefined` must be definitely assigned in the constructor.

```ts
class UserService {
  private repo: UserRepository;   // ❌ Property 'repo' has no initializer and is not
                                  //    definitely assigned in the constructor.
  async init(): Promise<void> {
    this.repo = new UserRepository();
  }
}
```

Four legitimate fixes, in order of preference:

```ts
// ✅ 1. Assign in the constructor (best — the object is never in a broken state):
class UserService {
  private readonly repo: UserRepository;
  constructor(db: DbConnection) {
    this.repo = new UserRepository(db);
  }
}

// ✅ 2. Parameter property (shorthand for the above — see 36 — parameter properties):
class UserService {
  constructor(private readonly repo: UserRepository) {}
}

// ✅ 3. Field initializer:
class RequestCounter {
  private counts = new Map<string, number>();
  private startedAt = new Date();
}

// ✅ 4. Definite assignment assertion — only when a framework assigns it for you:
class UserController {
  // Assigned by a DI container / decorator before any method runs:
  @Inject() private userService!: UserService;

  // Assigned by an async init() the framework guarantees to call first:
  private connection!: DbConnection;
  async onModuleInit(): Promise<void> {
    this.connection = await createConnection();
  }
}
```

`!:` carries the same contract as `!`: **you** guarantee it is assigned before any read. If a method runs before `onModuleInit`, you get `undefined` with no warning. Prefer options 1–3.

`!:` also works on `let` declarations, which is useful for try/catch assignment:

```ts
let connection!: DbConnection;      // no initializer, no "used before assigned" error
try {
  connection = await pool.connect();
} catch (err) {
  logger.error("Pool exhausted", { err });
  throw err;
}
await connection.query("SELECT 1");  // compiler is satisfied
```

### Concept 7 — `noUncheckedIndexedAccess`

This flag is **not** part of `strict: true`. You must enable it explicitly, and it is the natural completion of `strictNullChecks`.

Without it, TypeScript lies about every index access:

```ts
// noUncheckedIndexedAccess: false (default)
const userIds: number[] = [];
const firstId = userIds[0];
//    ^? number   ← a lie. The array is empty. This is undefined.
firstId.toFixed(2);            // ✅ compiles. 💥 at runtime.

const headers: Record<string, string> = {};
const contentType = headers["content-type"];
//    ^? string   ← a lie.
contentType.toLowerCase();     // ✅ compiles. 💥 at runtime.
```

With it on, every index access through a number index or index signature gains `| undefined`:

```ts
// noUncheckedIndexedAccess: true
const firstId = userIds[0];
//    ^? number | undefined   ← the truth
firstId.toFixed(2);            // ❌ 'firstId' is possibly 'undefined'.

const contentType = headers["content-type"];
//    ^? string | undefined   ← the truth

// You are forced to handle it:
if (contentType !== undefined && contentType.startsWith("application/json")) { /* ... */ }
// or:
const type = headers["content-type"] ?? "application/octet-stream";
```

What it does **not** change (deliberately):

```ts
// Declared properties are unaffected — only index signatures and numeric indexes:
interface Config { port: number }
declare const config: Config;
config.port;                   // number — still, correctly, not undefined

// for...of gives you elements, not indexed access — no undefined:
for (const userId of userIds) {
  userId.toFixed(0);           // ✅ number
}

// Array methods are unaffected:
userIds.map(id => id * 2);     // ✅ id: number

// Destructuring DOES get undefined (it is indexed access):
const [head] = userIds;
//     ^? number | undefined

// ⚠️ Writes are still allowed without a check:
userIds[10] = 5;               // ✅ fine — the flag only affects reads

// ⚠️ Tuples with known length are exempt at known indexes:
const pair: [string, number] = ["a", 1];
pair[0];                       // ✅ string — length is known
```

The cost is real: loops over arrays by index become noisy. The mitigation is to prefer `for...of`, `.entries()`, and array methods over `for (let i = 0; ...)`. That is better code anyway.

### Concept 8 — Narrowing patterns that actually work

```ts
declare const maybeUser: User | null | undefined;

// ── Truthiness — removes null AND undefined (and "" / 0 for primitives!) ────
if (maybeUser) { maybeUser.email; }                 // User

// ── Explicit null check — removes only null ─────────────────────────────────
if (maybeUser !== null) { maybeUser; }              // User | undefined

// ── Loose != null — removes BOTH null and undefined (the `==` special case) ─
if (maybeUser != null) { maybeUser.email; }         // User  ← useful, not a typo
// `x != null` is false only for null and undefined — this is the one place
// where `==` is genuinely the right operator.

// ── Early return / guard clause — narrows the REST of the function ──────────
function sendWelcome(user: User | null): void {
  if (!user) return;
  user.email;                                        // User for the whole rest
}

// ── typeof for primitives ──────────────────────────────────────────────────
declare const port: string | number | undefined;
if (typeof port === "number") { port.toFixed(0); }
if (typeof port === "string") { parseInt(port, 10); }

// ── ⚠️ Truthiness is WRONG for numbers and strings ──────────────────────────
function setRetryLimit(limit: number | undefined): number {
  if (!limit) return 3;      // ❌ limit === 0 also returns 3
  return limit;
}
function setRetryLimitFixed(limit: number | undefined): number {
  if (limit === undefined) return 3;   // ✅ 0 is preserved
  return limit;
}

// ── Narrowing an optional property — cache it in a local ────────────────────
interface Session { user?: User }
declare const session: Session;

if (session.user) {
  session.user.email;             // ✅ works — property narrowing on a const-ish access
}

// ⚠️ ...but narrowing is invalidated by any function call in between:
if (session.user) {
  await refreshSession(session);  // could have set session.user = undefined
  session.user.email;             // TS still says User — narrowing is NOT invalidated
                                  // by calls. This is a known unsoundness. Be careful.
}

// ✅ Safest: destructure into a local const, which cannot be reassigned:
const { user } = session;
if (user) {
  await refreshSession(session);
  user.email;                     // ✅ genuinely safe — `user` is a frozen local
}
```

### Concept 9 — Migrating an existing codebase

Turning `strictNullChecks: true` on a 200-file Node service produces four thousand errors and a strong urge to turn it back off. Do it incrementally:

```jsonc
// ── Step 0 — measure the damage without committing to anything ──────────────
// Run once with the flag on, count errors, do NOT commit:
//   npx tsc --noEmit --strictNullChecks
```

**Strategy A — leaf-first with a second tsconfig.** Keep the main build loose; add a strict config that only includes already-fixed files, and enforce it in CI:

```jsonc
// tsconfig.strict.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "strictNullChecks": true },
  "include": [
    "src/types/**/*.ts",
    "src/utils/**/*.ts",
    "src/domain/**/*.ts"    // ← grow this list one PR at a time
  ]
}
```

```jsonc
// package.json scripts
{
  "typecheck":        "tsc --noEmit -p tsconfig.json",
  "typecheck:strict": "tsc --noEmit -p tsconfig.strict.json"
}
```

**Strategy B — flip it on globally, suppress the rest.** Turn the flag on for everything, then add a suppression comment on each failing line and burn the list down:

```ts
// @ts-expect-error strictNullChecks migration — TICKET-4821
const email = user.email;
```

Use `@ts-expect-error`, never `@ts-ignore`: `@ts-expect-error` becomes an error itself once the underlying problem is fixed, so your suppressions self-clean. `@ts-ignore` rots forever.

**Order of work that minimizes churn:**

1. **Types and interfaces first** — mark nullable DB columns and optional request fields honestly. This is the source of truth; fixing it correctly makes downstream errors *meaningful* instead of noise.
2. **Repositories / data access** — `findById` returns `T | null`, `findAll` returns `T[]`.
3. **Domain / service layer** — this is where the real `if (!user) throw` guards belong.
4. **Controllers / routes** — mostly falls out for free once services are honest.
5. **Turn on `noUncheckedIndexedAccess` last**, as a separate project.

**Anti-pattern during migration:** mass-inserting `!` to silence errors. You will "finish" the migration with zero compile errors and exactly the same number of production crashes — plus a false sense of safety, which is worse than none. If you must bulk-suppress, use `@ts-expect-error` with a ticket number, because it is greppable and it expires.

---

## Example 1 — basic

```ts
// A small user lookup module, written the way strictNullChecks makes you write it.

interface User {
  id:          number;
  email:       string;
  displayName: string | null;   // nullable column — the type admits it
  lastLoginAt: Date | null;     // nullable column
}

// In-memory store standing in for a database:
const usersById = new Map<number, User>();

// ── Returning "not found" honestly ──────────────────────────────────────────
function findUserById(userId: number): User | null {
  // Map.get returns `User | undefined` — normalize to the null convention:
  return usersById.get(userId) ?? null;
}

// ── The caller CANNOT skip the check ────────────────────────────────────────
function getDisplayLabel(userId: number): string {
  const user = findUserById(userId);

  if (user === null) {
    return "Unknown user";        // ← the compiler required this branch
  }
  // user: User from here down

  // displayName is `string | null` — ?? handles it in one expression:
  return user.displayName ?? user.email;
}

// ── Optional parameters produce `| undefined`, not `| null` ─────────────────
function buildGreeting(user: User, salutation?: string): string {
  //                                ^? string | undefined
  const prefix = salutation ?? "Hello";
  const name   = user.displayName ?? user.email;
  return `${prefix}, ${name}!`;
}

// ── Dates: ?. gives you the method call only when the value exists ──────────
function describeLastLogin(user: User): string {
  const iso = user.lastLoginAt?.toISOString();
  //    ^? string | undefined
  return iso === undefined ? "Never logged in" : `Last seen ${iso}`;
}

// ── Array.find is `T | undefined` — always ──────────────────────────────────
function findUserByEmail(users: readonly User[], email: string): User | null {
  const match = users.find(u => u.email === email);
  //    ^? User | undefined
  return match ?? null;
}

// ── Truthiness vs === undefined: it matters for numbers ─────────────────────
function resolvePageSize(requested: number | undefined): number {
  // ❌ `requested || 20` would turn an explicit 0 into 20.
  if (requested === undefined) return 20;
  if (requested < 1)  return 1;
  if (requested > 100) return 100;
  return requested;
}

// ── Env vars are `string | undefined` under strictNullChecks ────────────────
function requireEnv(name: string): string {
  const value = process.env[name];
  if (value === undefined || value.trim() === "") {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;                    // string — a real guarantee, not an assertion
}

function optionalEnv(name: string, fallback: string): string {
  return process.env[name] ?? fallback;
}

const DATABASE_URL = requireEnv("DATABASE_URL");   // string
const LOG_LEVEL    = optionalEnv("LOG_LEVEL", "info");
const PORT         = Number(optionalEnv("PORT", "3000"));
```

---

## Example 2 — real world backend use case

```ts
// An Express-style request pipeline where every source of `undefined` is handled
// explicitly: headers, params, query, body, DB rows, JOINs, and env config.

import type { Request, Response, NextFunction } from "express";

// ── Domain types — nullability mirrors the database schema exactly ──────────

interface UserRow {
  id:            number;
  email:         string;
  password_hash: string;
  display_name:  string | null;   // NULL-able column
  deleted_at:    Date | null;     // NULL-able column (soft delete)
}

interface OrganizationRow {
  id:         number;
  name:       string;
  owner_id:   number;
  plan:       "free" | "pro" | "enterprise";
  trial_ends: Date | null;
}

interface AuthenticatedRequest extends Request {
  // Populated by the auth middleware. Optional because it is absent BEFORE
  // that middleware runs — this is honest, and it forces handlers to check.
  auth?: { userId: number; scopes: readonly string[] };
}

// ── Repositories — "may not exist" lives in the return type ─────────────────

class UserRepository {
  constructor(private readonly db: DbClient) {}

  async findById(userId: number): Promise<UserRow | null> {
    const rows = await this.db.query<UserRow>(
      "SELECT * FROM users WHERE id = $1 AND deleted_at IS NULL",
      [userId],
    );
    // rows[0] is `UserRow | undefined` under noUncheckedIndexedAccess:
    return rows[0] ?? null;
  }

  async findByEmail(email: string): Promise<UserRow | null> {
    const rows = await this.db.query<UserRow>(
      "SELECT * FROM users WHERE email = $1 AND deleted_at IS NULL",
      [email.toLowerCase()],
    );
    return rows[0] ?? null;
  }

  // A LEFT JOIN produces nullable columns even when the base row exists —
  // model that with a nullable nested object, not a flat bag of optionals:
  async findWithOrganization(
    userId: number,
  ): Promise<{ user: UserRow; organization: OrganizationRow | null } | null> {
    const rows = await this.db.query<UserRow & Partial<OrganizationRow>>(
      `SELECT u.*, o.id AS org_id, o.name AS org_name, o.owner_id, o.plan, o.trial_ends
         FROM users u
         LEFT JOIN organizations o ON o.owner_id = u.id
        WHERE u.id = $1`,
      [userId],
    );

    const row = rows[0];
    if (row === undefined) return null;

    const user: UserRow = {
      id:            row.id,
      email:         row.email,
      password_hash: row.password_hash,
      display_name:  row.display_name,
      deleted_at:    row.deleted_at,
    };

    // The JOIN produced no organization if the discriminating column is null:
    const organization: OrganizationRow | null =
      row.owner_id === undefined || row.owner_id === null
        ? null
        : {
            id:         row.id,
            name:       row.name,
            owner_id:   row.owner_id,
            plan:       row.plan ?? "free",
            trial_ends: row.trial_ends ?? null,
          };

    return { user, organization };
  }
}

// ── Config — every env read is checked exactly once, at boot ────────────────

interface AppConfig {
  readonly port:              number;
  readonly databaseUrl:       string;
  readonly jwtSecret:         string;
  readonly redisUrl:          string | null;   // genuinely optional feature
  readonly sentryDsn:         string | null;
  readonly rateLimitPerMin:   number;
  readonly enableSignups:     boolean;
}

function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  function required(name: string): string {
    const value = env[name];
    if (value === undefined || value.trim() === "") {
      throw new Error(`Missing required environment variable: ${name}`);
    }
    return value;
  }

  function optional(name: string): string | null {
    const value = env[name];
    return value === undefined || value.trim() === "" ? null : value;
  }

  function numeric(name: string, fallback: number): number {
    const raw = env[name];
    if (raw === undefined) return fallback;
    const parsed = Number(raw);
    if (Number.isNaN(parsed)) throw new Error(`Env var ${name} must be a number, got: ${raw}`);
    return parsed;
  }

  return {
    port:            numeric("PORT", 3000),
    databaseUrl:     required("DATABASE_URL"),
    jwtSecret:       required("JWT_SECRET"),
    redisUrl:        optional("REDIS_URL"),
    sentryDsn:       optional("SENTRY_DSN"),
    rateLimitPerMin: numeric("RATE_LIMIT_PER_MIN", 60),
    // `?? "true"` not `|| "true"` — an explicit "false" must survive:
    enableSignups:   (env.ENABLE_SIGNUPS ?? "true") === "true",
  };
}

// ── Middleware — headers are all `string | string[] | undefined` ────────────

function authenticate(jwtSecret: string) {
  return function (req: AuthenticatedRequest, res: Response, next: NextFunction): void {
    const authHeader = req.headers.authorization;
    //    ^? string | undefined

    if (authHeader === undefined) {
      res.status(401).json({ error: "Missing Authorization header", code: "NO_AUTH" });
      return;
    }

    // split() always returns an array, but its ELEMENTS are possibly undefined
    // under noUncheckedIndexedAccess:
    const [scheme, authToken] = authHeader.split(" ");
    //     ^? string | undefined, string | undefined

    if (scheme !== "Bearer" || authToken === undefined || authToken.length === 0) {
      res.status(401).json({ error: "Malformed Authorization header", code: "BAD_AUTH" });
      return;
    }

    const payload = verifyJwt(authToken, jwtSecret);   // returns JwtPayload | null
    if (payload === null) {
      res.status(401).json({ error: "Invalid or expired token", code: "BAD_TOKEN" });
      return;
    }

    req.auth = { userId: payload.sub, scopes: payload.scopes ?? [] };
    next();
  };
}

// ── Request parsing — params and query are all possibly undefined ───────────

interface ListOrdersQuery {
  readonly page:     number;
  readonly pageSize: number;
  readonly status:   "pending" | "shipped" | "cancelled" | null;
}

function parseListOrdersQuery(query: Request["query"]): ListOrdersQuery {
  // query values are `string | string[] | ParsedQs | ParsedQs[] | undefined`:
  function single(key: string): string | null {
    const value = query[key];
    if (typeof value === "string" && value.length > 0) return value;
    return null;
  }

  function positiveInt(key: string, fallback: number, max: number): number {
    const raw = single(key);
    if (raw === null) return fallback;
    const parsed = Number.parseInt(raw, 10);
    if (Number.isNaN(parsed) || parsed < 1) return fallback;
    return Math.min(parsed, max);
  }

  const statusRaw = single("status");
  const validStatuses = ["pending", "shipped", "cancelled"] as const;
  const status = validStatuses.find(s => s === statusRaw) ?? null;

  return {
    page:     positiveInt("page", 1, 10_000),
    pageSize: positiveInt("pageSize", 20, 100),
    status,
  };
}

// ── Service layer — the ONE place `null` becomes an HTTP error ──────────────

class ApiError extends Error {
  constructor(
    readonly statusCode: number,
    readonly code: string,
    message: string,
  ) {
    super(message);
  }
  static notFound(what: string): ApiError {
    return new ApiError(404, "NOT_FOUND", `${what} not found`);
  }
  static unauthorized(message = "Unauthorized"): ApiError {
    return new ApiError(401, "UNAUTHORIZED", message);
  }
}

interface UserProfileDto {
  readonly userId:           number;
  readonly email:            string;
  readonly displayName:      string;         // never null in the DTO — resolved
  readonly organizationName: string | null;  // genuinely absent for solo users
  readonly plan:             "free" | "pro" | "enterprise";
  readonly trialDaysLeft:    number | null;
}

class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly clock:    () => Date,
  ) {}

  async getProfile(userId: number): Promise<UserProfileDto> {
    const record = await this.userRepo.findWithOrganization(userId);
    if (record === null) throw ApiError.notFound("User");
    // record: { user: UserRow; organization: OrganizationRow | null }

    const { user, organization } = record;

    // Resolve the nullable column into a guaranteed value at the API boundary —
    // the DTO promises `string`, so the null dies here and nowhere else:
    const displayName = user.display_name ?? user.email.split("@")[0] ?? user.email;

    const trialEnds = organization?.trial_ends ?? null;
    const trialDaysLeft =
      trialEnds === null
        ? null
        : Math.max(0, Math.ceil((trialEnds.getTime() - this.clock().getTime()) / 86_400_000));

    return {
      userId:           user.id,
      email:            user.email,
      displayName,
      organizationName: organization?.name ?? null,
      plan:             organization?.plan ?? "free",
      trialDaysLeft,
    };
  }
}

// ── Handler — `req.auth` is optional, so the check is mandatory ─────────────

function makeGetProfileHandler(userService: UserService) {
  return async function (
    req: AuthenticatedRequest,
    res: Response,
    next: NextFunction,
  ): Promise<void> {
    try {
      const auth = req.auth;
      if (auth === undefined) {
        // Only reachable if the route was wired without the auth middleware —
        // strictNullChecks turned a silent 500 into a handled 401.
        throw ApiError.unauthorized("Authentication middleware not applied");
      }

      const idParam = req.params.id;
      //    ^? string | undefined
      if (idParam === undefined) throw new ApiError(400, "BAD_REQUEST", "Missing :id param");

      const targetUserId = Number.parseInt(idParam, 10);
      if (Number.isNaN(targetUserId)) {
        throw new ApiError(400, "BAD_REQUEST", `Invalid user id: ${idParam}`);
      }

      const profile = await userService.getProfile(targetUserId);
      res.status(200).json({ data: profile });
    } catch (err) {
      next(err);
    }
  };
}

// ── Declarations used above ─────────────────────────────────────────────────
interface JwtPayload { sub: number; scopes?: readonly string[] }
declare function verifyJwt(token: string, secret: string): JwtPayload | null;
interface DbClient { query<T>(sql: string, params: readonly unknown[]): Promise<T[]> }
```

---

## Going deeper

### The compiler removes `undefined` from optional properties only under `exactOptionalPropertyTypes`

By default, `field?: string` means `string | undefined`, and you are allowed to *explicitly assign* `undefined`:

```ts
interface UpdateUserPatch { displayName?: string }

const patch: UpdateUserPatch = { displayName: undefined };  // ✅ allowed by default
"displayName" in patch;                                      // true — the KEY exists!
```

That is a real bug source: code that checks `in` sees the key, code that checks `!== undefined` does not. `exactOptionalPropertyTypes: true` (not in `strict`) separates the two:

```ts
// exactOptionalPropertyTypes: true
const patch: UpdateUserPatch = { displayName: undefined };
// ❌ Type 'undefined' is not assignable to type 'string' with
//    'exactOptionalPropertyTypes: true'. Consider adding 'undefined' to the type.

const ok1: UpdateUserPatch = {};                             // ✅ key absent
interface Explicit { displayName?: string | undefined }       // opt into both
const ok2: Explicit = { displayName: undefined };             // ✅
```

Enable it if you build PATCH-style APIs where "absent" and "explicitly cleared" differ. Skip it otherwise — it is noisy with third-party types.

### Narrowing is not invalidated by function calls — a known unsoundness

```ts
interface Connection { socket: Socket | null }
declare const conn: Connection;
declare function reset(c: Connection): void;   // sets c.socket = null

if (conn.socket !== null) {
  reset(conn);
  conn.socket.write("ping");   // ✅ compiles. 💥 at runtime.
  //   ^? Socket   ← TypeScript does not re-check after the call
}
```

TypeScript deliberately does not invalidate narrowing across calls — doing so correctly would require whole-program aliasing analysis and would make almost all narrowing useless. The mitigation is the same one that fixes half of all concurrency bugs: **copy into a local `const`**.

```ts
const { socket } = conn;
if (socket !== null) {
  reset(conn);
  socket.write("ping");        // ✅ genuinely safe — `socket` cannot be reassigned
}
```

The same applies to `await`. Any narrowing of a mutable object property established before an `await` is not guaranteed after it, but the compiler still believes it.

### Narrowing is lost inside closures over `let`

```ts
let currentUser: User | null = await findUser(userId);

if (currentUser !== null) {
  setTimeout(() => {
    currentUser.email;   // ❌ 'currentUser' is possibly 'null'.
  }, 1000);
}
```

The compiler cannot know when the callback runs, and `currentUser` is a `let` that could be reassigned in between. `const` fixes it, because a `const` can never change after narrowing:

```ts
const currentUser = await findUser(userId);
if (currentUser !== null) {
  setTimeout(() => {
    currentUser.email;   // ✅ const narrowing survives into the closure
  }, 1000);
}
```

This is the strongest practical argument for `const` by default in TypeScript that does not exist in JavaScript at all.

### `Array.prototype.filter` does not narrow — unless you help it

```ts
const maybeUsers: (User | null)[] = await Promise.all(ids.map(findUserById));

const users = maybeUsers.filter(u => u !== null);
//    ^? (User | null)[]   ← in TS < 5.5, the null survives!
```

TypeScript 5.5+ has **inferred type predicates** and will narrow this correctly for simple, single-expression callbacks. On older versions (or complex callbacks) you must supply a type predicate:

```ts
function isNotNull<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}

const users = maybeUsers.filter(isNotNull);
//    ^? User[]   ✅ on every TS version
```

Note the constraint on inferred predicates: the callback must be a single `return` expression with no assignments, and the parameter must not be reassigned. `filter(u => { const x = u; return x !== null; })` will *not* infer a predicate.

### `??` and `?.` compile down, `!` compiles away — check your target

```ts
const host = overrideHost ?? "localhost";
const id   = org.billing?.id;
const user = findUser(1)!;
```

With `"target": "ES2020"` or higher, `??` and `?.` emit as-is (they are native). With `"target": "ES2019"` or lower, they downlevel into temp-variable ladders, which is a small but nonzero bundle and perf cost in hot loops:

```js
// ES2019 output of `org.billing?.defaultCard?.last4`:
var _a, _b;
(_b = (_a = org.billing) === null || _a === void 0 ? void 0 : _a.defaultCard) === null || _b === void 0 ? void 0 : _b.last4;
```

`!` emits nothing at all — `findUser(1)!` becomes `findUser(1)`. That is precisely why it is dangerous: there is no artifact of it in the running program.

### `strictNullChecks` changes inference, not just checking

The flag changes what types are *inferred*, which can surprise you:

```ts
// strictNullChecks: false
let cursor = null;         // cursor: any
cursor = "abc";            // fine

// strictNullChecks: true
let cursor = null;         // cursor: any   ← still any! (special "auto" widening)
cursor = "abc";            // fine — TS widens `null` initializers to any
                           // ...but only WITHOUT a type annotation and only for `let`.

const frozen = null;       // frozen: null  ← const does not widen
```

More importantly, the *return type* of a function with no explicit annotation changes:

```ts
function findFirst<T>(items: T[], pred: (t: T) => boolean) {
  for (const item of items) if (pred(item)) return item;
  // no explicit return at the end
}
// strictNullChecks: false → returns T
// strictNullChecks: true  → returns T | undefined   ← the implicit undefined is now visible
```

This is why flipping the flag can produce errors in files you did not change: their *callers* now see honest types.

### `null` and `undefined` in `Record` and index signatures

`Record<string, User>` claims every string key maps to a `User`. That is never true:

```ts
const sessionsByToken: Record<string, Session> = {};
sessionsByToken["nonexistent"].userId;   // ✅ compiles. 💥 at runtime.
```

Three fixes, best last:

```ts
// 1. Be honest in the type:
const sessions: Record<string, Session | undefined> = {};
sessions["x"]?.userId;                   // ✅ forced to check

// 2. Enable noUncheckedIndexedAccess — applies to ALL index signatures at once.

// 3. Use a Map — get() has ALWAYS returned `V | undefined`, flag or not:
const sessionMap = new Map<string, Session>();
sessionMap.get("x")?.userId;             // ✅ correct by construction
```

`Map` is the underrated winner here: its type signature was honest before `strictNullChecks` existed.

### `void` vs `undefined` — they are not the same

```ts
function noop(): void {}
function returnsUndefined(): undefined { return undefined; }

const a: undefined = noop();            // ❌ Type 'void' is not assignable to 'undefined'
const b: void = returnsUndefined();     // ✅ undefined IS assignable to void

// void in a callback position means "I ignore your return value":
type Callback = (userId: number) => void;
const cb: Callback = (userId) => userId * 2;   // ✅ returning number is fine
```

`void` is "the return value is meaningless", `undefined` is "the return value is the undefined value". Use `void` for callbacks and procedures; use `undefined` only when a caller might meaningfully compare against it.

### Performance of the flag itself

`strictNullChecks` makes type checking **slower** — often 10–30% on large projects — because the compiler tracks control-flow-based narrowing through every branch. It has zero effect on emitted JavaScript. If `tsc` gets slow after enabling it, the fix is `incremental: true` plus project references (see `03 — tsconfig in depth`), not turning the flag back off.

---

## Common mistakes

### Mistake 1 — Reaching for `!` instead of a real check

```ts
// ❌ Wrong — the assertion does nothing at runtime; this crashes on bad input:
async function deleteUser(req: Request, res: Response): Promise<void> {
  const userId = Number(req.params.id!);
  const user   = (await userRepo.findById(userId))!;
  await userRepo.delete(user.id);          // 💥 "Cannot read properties of null"
  res.status(204).send();
}

// ✅ Right — each absent value produces a correct HTTP response:
async function deleteUser(req: Request, res: Response): Promise<void> {
  const idParam = req.params.id;
  if (idParam === undefined) {
    res.status(400).json({ error: "Missing user id" });
    return;
  }

  const userId = Number.parseInt(idParam, 10);
  if (Number.isNaN(userId)) {
    res.status(400).json({ error: `Invalid user id: ${idParam}` });
    return;
  }

  const user = await userRepo.findById(userId);
  if (user === null) {
    res.status(404).json({ error: "User not found" });
    return;
  }

  await userRepo.delete(user.id);          // ✅ user is genuinely a User
  res.status(204).send();
}
```

### Mistake 2 — Using `||` where you meant `??`

```ts
interface PaginationOptions { offset?: number; limit?: number; includeDeleted?: boolean }

// ❌ Wrong — offset 0 becomes 50, includeDeleted:false becomes true:
function paginate(opts: PaginationOptions) {
  const offset         = opts.offset || 50;
  const limit          = opts.limit  || 20;
  const includeDeleted = opts.includeDeleted || true;   // ALWAYS true. Always.
  return { offset, limit, includeDeleted };
}
paginate({ offset: 0, includeDeleted: false });
// → { offset: 50, limit: 20, includeDeleted: true }   ← every value wrong

// ✅ Right — ?? only substitutes for null/undefined:
function paginate(opts: PaginationOptions) {
  const offset         = opts.offset         ?? 0;
  const limit          = opts.limit          ?? 20;
  const includeDeleted = opts.includeDeleted ?? false;
  return { offset, limit, includeDeleted };
}
paginate({ offset: 0, includeDeleted: false });
// → { offset: 0, limit: 20, includeDeleted: false }   ✅
```

### Mistake 3 — Declaring types as non-nullable when the data is nullable

```ts
// ❌ Wrong — the interface claims these are always present, so the compiler
//    NEVER warns you, and strictNullChecks buys you nothing:
interface User {
  id:          number;
  email:       string;
  displayName: string;    // but the DB column is `display_name TEXT NULL`
  deletedAt:   Date;      // but the DB column is `deleted_at TIMESTAMP NULL`
}

function renderUser(user: User): string {
  return user.displayName.toUpperCase();   // ✅ compiles. 💥 for half your rows.
}

// ✅ Right — the type mirrors the schema; the compiler does its job:
interface User {
  id:          number;
  email:       string;
  displayName: string | null;
  deletedAt:   Date | null;
}

function renderUser(user: User): string {
  return (user.displayName ?? user.email).toUpperCase();   // ✅ handled
}
```

The rule: **strictNullChecks only protects you as far as your type declarations are honest.** A lying interface is a lying interface with or without the flag.

### Mistake 4 — `?.` on the wrong link in the chain

```ts
interface Organization { billing: BillingAccount | null }
interface BillingAccount { defaultCard: Card | null; stripeId: string }
interface Card { last4: string }

declare const org: Organization;

// ❌ Wrong — protects `billing` but then dereferences a possibly-null card:
const last4 = org.billing?.defaultCard.last4;
// ❌ 'org.billing.defaultCard' is possibly 'null'.
// (and if it compiled, it would crash whenever billing exists but card is null)

// ❌ Also wrong — ?. everywhere, hiding real bugs:
const id = org?.billing?.stripeId;
//         ^ `org` is not nullable — this ?. is noise that hides future changes

// ✅ Right — one ?. per genuinely nullable link, none anywhere else:
const last4 = org.billing?.defaultCard?.last4 ?? "----";
const id    = org.billing?.stripeId ?? null;
```

### Mistake 5 — `strictPropertyInitialization` silenced with `!:` everywhere

```ts
// ❌ Wrong — every field asserted; the object is constructible in a broken state:
class OrderService {
  private orderRepo!:  OrderRepository;
  private paymentSvc!: PaymentService;
  private logger!:     Logger;
  // Nothing assigns these. `new OrderService().placeOrder()` → 💥
}

// ✅ Right — constructor injection makes "broken instance" unrepresentable:
class OrderService {
  constructor(
    private readonly orderRepo:  OrderRepository,
    private readonly paymentSvc: PaymentService,
    private readonly logger:     Logger,
  ) {}
}

// ✅ Acceptable use — a value a framework provably assigns before any method:
class OrderController {
  private connection!: DbConnection;         // assigned in onModuleInit()
  async onModuleInit(): Promise<void> {
    this.connection = await createConnection();
  }
}
```

### Mistake 6 — Assuming `noUncheckedIndexedAccess` protects writes or `.length`

```ts
const queue: Task[] = [];

// ❌ Wrong assumption — the flag does not stop out-of-bounds writes:
queue[5] = task;                    // ✅ compiles — creates a sparse array with holes
queue.length;                       // 6 — and queue[0..4] are all undefined at runtime
//    but their TYPE is `Task`... unless you read them, which gives `Task | undefined`

// ❌ Wrong — length check does not narrow the index access:
if (queue.length > 0) {
  queue[0].run();                   // ❌ 'queue[0]' is possibly 'undefined'
}

// ✅ Right — read once into a local and check the value, not the length:
const first = queue[0];
if (first !== undefined) {
  first.run();                      // ✅ Task
}

// ✅ Or use methods that model emptiness properly:
const next = queue.shift();         // Task | undefined — always was
if (next !== undefined) next.run();

// ✅ Or iterate, which never produces undefined:
for (const task of queue) task.run();
```

---

## Practice exercises

### Exercise 1 — easy

Write a small "user settings resolver" module for a Node service, with `strictNullChecks` and `noUncheckedIndexedAccess` both enabled.

Define:

- `interface UserSettings` with `theme: "light" | "dark"`, `timezone: string`, `emailDigest: boolean`, `weeklyReportDay: 0|1|2|3|4|5|6 | null`
- `interface UserSettingsPatch` — every field optional, where `weeklyReportDay` may also be explicitly `null`

Then implement:

1. `defaultSettings(): UserSettings` — sensible defaults (`"light"`, `"UTC"`, `true`, `null`)
2. `mergeSettings(current: UserSettings, patch: UserSettingsPatch): UserSettings` — must use `??`, not `||`, and must correctly preserve `emailDigest: false` and `weeklyReportDay: null`
3. `getFirstAdminEmail(emails: readonly string[]): string | null` — returns `emails[0]` safely, no `!`
4. `describeReportDay(settings: UserSettings): string` — `"No weekly report"` when null, otherwise the day name; use a `readonly string[]` lookup table and handle the possibly-undefined index read without `!`

Constraint: your solution must contain **zero** `!` non-null assertions.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a fully null-safe environment configuration loader for a backend service.

Requirements:

- `interface ServiceConfig` with: `port: number`, `nodeEnv: "development" | "test" | "production"`, `databaseUrl: string`, `jwtSecret: string`, `jwtExpirySeconds: number`, `redisUrl: string | null`, `smtp: { host: string; port: number; user: string; password: string } | null` (the whole SMTP block is optional — but if *any* SMTP var is present, *all* must be), `corsOrigins: readonly string[]`, `enableMetrics: boolean`
- `loadConfig(env: NodeJS.ProcessEnv): ServiceConfig` that:
  - throws a single aggregated `ConfigError` listing **all** missing/invalid variables, not just the first one
  - never uses `!` or `as`
  - treats empty strings as absent
  - parses `PORT`, `JWT_EXPIRY_SECONDS`, `SMTP_PORT` as integers and rejects `NaN` and non-positive values
  - parses `CORS_ORIGINS` as a comma-separated list, trimming entries and dropping empties, defaulting to `[]`
  - parses `ENABLE_METRICS` such that the string `"false"` yields `false` (a `||` bug here must be impossible)
  - validates `NODE_ENV` against the three allowed literals, defaulting to `"development"`
  - implements the all-or-nothing SMTP rule: if zero SMTP vars are set → `smtp: null`; if some but not all → a `ConfigError` naming the missing ones

Then write `assertConfigLoads()` style test cases (plain functions, no test framework) covering: everything present, nothing optional present, partial SMTP, `PORT=0`, `ENABLE_METRICS=false`, and `CORS_ORIGINS=" a , ,b "`.

```ts
// Write your code here
```

### Exercise 3 — hard

Migrate a deliberately loose module to full strict-null safety, then extend it.

Part A — you are given this (pretend) legacy signature set. Rewrite every type so nullability is honest, then rewrite every function body so it compiles under `strictNullChecks: true`, `strictPropertyInitialization: true`, and `noUncheckedIndexedAccess: true`, with zero `!` and zero `as`:

```ts
// LEGACY — do not keep these signatures, fix them:
// interface Order { id: number; userId: number; couponCode: string; shippedAt: Date; items: OrderItem[] }
// interface OrderItem { sku: string; qty: number; unitPriceCents: number; discountCents: number }
// class OrderRepo { findById(id: number): Order; findByUser(userId: number): Order[] }
// class CouponRepo { find(code: string): Coupon }
// function calculateTotal(order: Order, coupon: Coupon): number
// function summarize(orders: Order[]): { total: number; firstOrderDate: Date; lastOrderDate: Date }
```

Notes: `couponCode` and `shippedAt` are nullable columns; both repo lookups can miss; `summarize` on an empty array has no first/last date.

Part B — build a `NullSafeCache<K, V>` class with:

- `get(key: K): V | undefined`
- `getOrThrow(key: K, onMissing: () => Error): V`
- `getOrCompute(key: K, compute: () => V): V` — using `??=` semantics, must not recompute on a cached falsy value like `0` or `""`
- `getMany(keys: readonly K[]): { found: Map<K, V>; missing: K[] }`
- a `stats` field that is definitely initialized without `!:`

Part C — write `assertNonNull<T>(value: T | null | undefined, message: string): T` that performs a **real** runtime check and returns the narrowed value, and then explain in comments why this is strictly better than `value!` at every single call site, including what happens in each version when the value actually is missing.

Part D — write two versions of a `processPendingOrders(orderIds: readonly number[])` function: one that uses `Array.prototype.filter` with a hand-written type predicate to drop missing orders, and one that uses a `for...of` loop with explicit `continue`. Comment on which one survives a TypeScript downgrade below 5.5 and why.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Enable it ───────────────────────────────────────────────────────────────
// tsconfig.json
// { "compilerOptions": {
//     "strict": true,                      // includes strictNullChecks +
//                                          // strictPropertyInitialization
//     "noUncheckedIndexedAccess": true,    // NOT in strict — enable separately
//     "exactOptionalPropertyTypes": true   // NOT in strict — optional, strictest
// } }

// ── Declaring nullability ───────────────────────────────────────────────────
let a: string | null;         // may be null
let b: string | undefined;    // may be undefined
interface X { c?: string }    // optional key → string | undefined
interface Y { d: string | null }  // required key, nullable value

// ── Narrowing ───────────────────────────────────────────────────────────────
if (v)              { }       // removes null, undefined, "", 0, false, NaN
if (v !== null)     { }       // removes null only
if (v !== undefined){ }       // removes undefined only
if (v != null)      { }       // removes BOTH null and undefined
if (!v) return;               // guard clause narrows the rest of the function
const { v } = obj;            // copy to const → narrowing survives closures & awaits

// ── Operators ───────────────────────────────────────────────────────────────
obj?.prop                     // undefined if obj is null/undefined
obj?.[key]                    // safe index
fn?.(arg)                     // safe call
a ?? b                        // b only when a is null/undefined
a ||= b                       // assign when a is falsy
a ??= b                       // assign when a is null/undefined
v!                            // strip null|undefined — NO runtime check
let v!: T                     // definite assignment assertion
class C { p!: T }             // property definite assignment assertion

// ── Safe patterns ───────────────────────────────────────────────────────────
function requireEnv(name: string): string {
  const value = process.env[name];
  if (value === undefined || value === "") throw new Error(`Missing env: ${name}`);
  return value;
}
function isNotNull<T>(v: T | null | undefined): v is T { return v != null; }
const users = maybeUsers.filter(isNotNull);            // (User|null)[] → User[]
const first = arr[0]; if (first !== undefined) { }     // noUncheckedIndexedAccess
map.get(k) ?? fallback;                                // Map.get is honest already
```

| Thing | Meaning | Runtime check? |
|---|---|---|
| `strictNullChecks` | `null`/`undefined` are separate types, not in every type | n/a (compile only) |
| `strictPropertyInitialization` | class fields must be definitely assigned | no |
| `noUncheckedIndexedAccess` | `arr[i]` / `rec[k]` reads gain `\| undefined` | no |
| `exactOptionalPropertyTypes` | `k?: T` forbids explicitly assigning `undefined` | no |
| `v?.p` | short-circuit to `undefined` on null/undefined | **yes** |
| `a ?? b` | fallback on null/undefined only | **yes** |
| `a \|\| b` | fallback on any falsy value — usually a bug | **yes** |
| `v!` | erase `null\|undefined` from the type | **no — emits nothing** |
| `let v!: T` / `p!: T` | "assigned before use, trust me" | **no** |
| `v != null` | the one correct use of `==` — excludes both | **yes** |
| `undefined` (convention) | never provided / absent key | — |
| `null` (convention) | present and explicitly empty (SQL NULL, JSON null) | — |

---

## Connected topics

- **07 — Special types** — where `null`, `undefined`, `void`, `never`, and `unknown` come from and how they relate.
- **09 — Union types** — `User | null` is just a union; every narrowing rule for unions applies here.
- **40 — Type narrowing** — the control-flow analysis that turns `User | null` into `User`, and where it breaks down.
- **41 — Type guards** — writing `isNotNull<T>(v): v is T` so `.filter()` narrows on every TS version.
- **03 — tsconfig in depth** — where `strict`, `noUncheckedIndexedAccess`, and `exactOptionalPropertyTypes` live and how project references keep checking fast.
