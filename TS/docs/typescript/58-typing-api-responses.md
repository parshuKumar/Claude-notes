# 58 — Typing API Responses

## What is this?

**Typing API responses** means defining *one* canonical shape that every HTTP endpoint in your service returns, expressing that shape as a TypeScript type, and then wiring it through your handlers, your helpers, and (ideally) your client — so that the compiler, not code review, is what guarantees your API is consistent.

The core artifact is a single generic discriminated union:

```ts
// One envelope for the whole application:
type ApiResponse<T> =
  | { success: true;  data: T }
  | { success: false; error: ApiError };
```

Everything else in this doc is built on top of that one type: typed error codes, paginated payloads, `res.json()` that refuses wrong shapes, `ok()` / `fail()` builders, DTO types that structurally cannot leak `passwordHash`, and a shared package so the frontend gets the same types the backend produces.

---

## Why does it matter?

An untyped Node API drifts. Not maliciously — just by accretion:

- `/users/:id` returns `{ user }`.
- `/users` returns `{ data: [], total }`.
- `/users/:id/orders` returns a bare array.
- The 404 path returns `{ error: "not found" }`, the 500 path returns `{ message: "..." }`, and the validation middleware returns `{ errors: [...] }`.
- Someone adds `res.json(user)` in a hurry and ships `passwordHash` to every browser on the internet.

Now every client is a pile of special cases. Every error handler checks three field names. Every refactor is a manual audit. And the one bug that actually matters — leaking a secret column — is invisible because nothing describes what an endpoint is *allowed* to return.

A typed response envelope fixes all of that at once:

- **One shape** — clients write exactly one unwrapping function.
- **One error contract** — `error.code` is a union, so a client `switch` is exhaustive.
- **Compile-time leak protection** — if `UserDto` has no `passwordHash` field, `res.json(ok(user))` where `user` is a domain `User` is a *type error*, not a security incident.
- **Free client types** — export the types, import them in the frontend, delete your hand-written API interfaces.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: every handler invents its own contract ──────────────────────

// routes/users.js
app.get("/users/:id", async (req, res) => {
  const user = await userRepo.findById(req.params.id);
  if (!user) {
    return res.status(404).json({ error: "User not found" });  // shape A
  }
  res.json(user);              // shape B — and it includes passwordHash. Nobody noticed.
});

app.get("/users", async (req, res) => {
  const users = await userRepo.list();
  res.json({ data: users, total: users.length });               // shape C
});

app.post("/users", async (req, res) => {
  try {
    const user = await userService.create(req.body);
    res.status(201).json({ user });                             // shape D
  } catch (err) {
    res.status(500).json({ message: err.message });             // shape E
  }
});

// Client code is now defensive guesswork:
const body = await fetch("/users/1").then((r) => r.json());
const user = body.user ?? body.data ?? body;      // pick whichever exists
const errText = body.error ?? body.message ?? body.errors?.[0]?.msg ?? "Unknown";
// Nothing tells you which endpoint returns which. You find out in production.
```

```ts
// ── TypeScript: one envelope, enforced by the compiler ──────────────────────

// One union for the whole app — success XOR failure, never both:
type ApiResponse<T> =
  | { success: true;  data: T }
  | { success: false; error: ApiErrorBody };

// One error body, with a CLOSED set of machine-readable codes:
interface ApiErrorBody {
  code:    ApiErrorCode;          // "NOT_FOUND" | "VALIDATION_FAILED" | ...
  message: string;
  details?: unknown;
}

// A DTO that structurally cannot carry secrets:
interface UserDto {
  userId:    string;
  email:     string;
  fullName:  string;
  createdAt: string;              // ISO string — not a Date, JSON has no Date
}
// note: no passwordHash field exists here, so it cannot be assigned in.

app.get("/users/:userId", async (req, res: Response<ApiResponse<UserDto>>) => {
  const user = await userRepo.findById(req.params.userId);   // domain User, has passwordHash
  if (!user) {
    return res.status(404).json(fail("NOT_FOUND", "User not found"));
  }
  res.json(ok(toUserDto(user)));   // ✅ must go through the DTO mapper — compiler insists
  // res.json(ok(user));           // ❌ Error: 'passwordHash' is not in type 'UserDto'
});

// Client code, with zero guesswork:
const body: ApiResponse<UserDto> = await fetch("/users/1").then((r) => r.json());
if (body.success) {
  body.data.email;      // ✅ UserDto — autocomplete, no optional chaining
} else {
  body.error.code;      // ✅ a union — a switch over it can be exhaustive
}
```

The revelation: the envelope is not extra ceremony. It is the thing that lets you delete all the defensive code on both sides of the wire, *and* it is the only place where "which fields may leave this process" is written down.

---

## Syntax

```ts
// ── The envelope: a generic discriminated union ─────────────────────────────
type ApiResponse<T> =
  | { success: true;  data: T }                    // success branch carries the payload
  | { success: false; error: ApiErrorBody };       // failure branch carries the error

// ── Closed union of machine-readable error codes ───────────────────────────
type ApiErrorCode =
  | "VALIDATION_FAILED"                            // 422 — bad requestBody
  | "UNAUTHENTICATED"                              // 401 — missing/invalid authToken
  | "NOT_FOUND";                                   // 404 — resource absent

// ── The error body shape ───────────────────────────────────────────────────
interface ApiErrorBody {
  code:     ApiErrorCode;                          // stable, machine-branchable
  message:  string;                                // human-readable, may change freely
  details?: unknown;                               // optional per-code extra payload
}

// ── Builders — the ONLY way responses get constructed ──────────────────────
const ok = <T>(data: T): ApiResponse<T> =>
  ({ success: true, data });                       // `success: true` inferred as literal via the return type

const fail = (code: ApiErrorCode, message: string): ApiResponse<never> =>
  ({ success: false, error: { code, message } });  // ApiResponse<never> assigns to ApiResponse<anything>

// ── Typed res.json — Express's Response is generic in the body type ────────
import type { Request, Response } from "express";

app.get("/health", (_req: Request, res: Response<ApiResponse<{ uptime: number }>>) => {
  res.json(ok({ uptime: process.uptime() }));      // ✅ matches
  // res.json({ status: "up" });                   // ❌ compile error — wrong shape
});
```

---

## How it works — concept by concept

### Concept 1 — One envelope, applied everywhere

The value of an envelope is proportional to how universally it is applied. One endpoint that returns a bare array undoes most of the benefit, because clients must now know *which* endpoints are enveloped.

```ts
// src/http/response.ts — the single source of truth for your wire format.

/** Every HTTP body your service emits is one of these two shapes. */
export type ApiResponse<T> =
  | ApiSuccess<T>
  | ApiFailure;

export interface ApiSuccess<T> {
  readonly success: true;
  readonly data:    T;
  /** Correlates the response with the server log line for this request. */
  readonly requestId: string;
}

export interface ApiFailure {
  readonly success: false;
  readonly error:   ApiErrorBody;
  readonly requestId: string;
}

export interface ApiErrorBody {
  readonly code:     ApiErrorCode;
  readonly message:  string;
  readonly details?: unknown;
}
```

Because `success` is a **boolean literal** (`true` / `false`, not `boolean`), this is a discriminated union — see `42 — Discriminated unions`. A client that checks `if (body.success)` gets `data` and nothing else; the `else` branch gets `error` and nothing else. It is impossible to write a handler that reads `body.data` on a failure.

```ts
// Consuming it — narrowing does all the work:
function unwrap<T>(body: ApiResponse<T>): T {
  if (body.success) {
    return body.data;                        // body: ApiSuccess<T>
  }
  throw new ApiClientError(body.error.code, body.error.message);  // body: ApiFailure
}
```

### Concept 2 — Typed error codes as a closed union

The single highest-leverage decision in an API contract is making the error code a **closed union of string literals** rather than `string`.

```ts
export type ApiErrorCode =
  // ── 4xx — the caller's fault ─────────────────────────────────────────────
  | "BAD_REQUEST"          // 400 — malformed request, not a field-level problem
  | "UNAUTHENTICATED"      // 401 — authToken missing, expired, or unparseable
  | "FORBIDDEN"            // 403 — authenticated, but not permitted
  | "NOT_FOUND"            // 404 — resource does not exist (or is hidden from caller)
  | "CONFLICT"             // 409 — unique constraint, optimistic-lock failure
  | "VALIDATION_FAILED"    // 422 — requestBody parsed but failed schema checks
  | "RATE_LIMITED"         // 429 — too many requests from this principal
  // ── 5xx — our fault ──────────────────────────────────────────────────────
  | "INTERNAL"             // 500 — unexpected; never expose internals in `message`
  | "UPSTREAM_UNAVAILABLE";// 503 — a dependency (db, payment gateway) is down

// Pair each code with its HTTP status ONCE — a Record is exhaustive by construction.
// Add a code to the union above and this object becomes a compile error until you
// give it a status. (See `43 — Mapped types` / `32 — Utility types` for Record.)
export const STATUS_FOR_CODE: Record<ApiErrorCode, number> = {
  BAD_REQUEST:          400,
  UNAUTHENTICATED:      401,
  FORBIDDEN:            403,
  NOT_FOUND:            404,
  CONFLICT:             409,
  VALIDATION_FAILED:    422,
  RATE_LIMITED:         429,
  INTERNAL:             500,
  UPSTREAM_UNAVAILABLE: 503,
};
```

Now a client can branch exhaustively:

```ts
function describeFailure(error: ApiErrorBody): string {
  switch (error.code) {
    case "UNAUTHENTICATED":   return "Please sign in again.";
    case "FORBIDDEN":         return "You do not have access to this resource.";
    case "NOT_FOUND":         return "That item no longer exists.";
    case "CONFLICT":          return "Someone else changed this first — reload and retry.";
    case "VALIDATION_FAILED": return "Please fix the highlighted fields.";
    case "RATE_LIMITED":      return "Slow down — try again in a minute.";
    case "BAD_REQUEST":
    case "INTERNAL":
    case "UPSTREAM_UNAVAILABLE":
      return "Something went wrong on our side.";
    default: {
      const _exhaustive: never = error.code;   // ← compile error if a code is added
      throw new Error(`Unhandled error code: ${String(_exhaustive)}`);
    }
  }
}
```

Adding `"PAYMENT_REQUIRED"` to `ApiErrorCode` breaks the build in exactly two places: `STATUS_FOR_CODE` and this switch. That is the whole point.

### Concept 3 — Per-code `details` via a discriminated error union

`details?: unknown` is honest but weak. If some codes carry structured extras, make the *error body itself* a discriminated union keyed on `code`:

```ts
export type ApiErrorBody =
  | { code: "VALIDATION_FAILED"; message: string; details: FieldIssue[] }
  | { code: "RATE_LIMITED";      message: string; details: { retryAfterSeconds: number } }
  | { code: "CONFLICT";          message: string; details: { conflictingField: string } }
  | { code: Exclude<ApiErrorCode, "VALIDATION_FAILED" | "RATE_LIMITED" | "CONFLICT">;
      message: string; details?: undefined };

export interface FieldIssue {
  readonly field:   string;      // "requestBody.email"
  readonly message: string;      // "must be a valid email address"
}

// The client now gets exact types per branch:
function renderError(error: ApiErrorBody): string[] {
  if (error.code === "VALIDATION_FAILED") {
    return error.details.map((issue) => `${issue.field}: ${issue.message}`);  // ✅ FieldIssue[]
  }
  if (error.code === "RATE_LIMITED") {
    return [`Retry in ${error.details.retryAfterSeconds}s`];                  // ✅ number
  }
  return [error.message];                                                      // details is undefined here
}
```

The `Exclude<...>` catch-all member keeps the union from having to list every plain code by hand — see `32 — Utility types`.

### Concept 4 — Response builders (`ok`, `fail`) as the only constructors

Never construct an envelope inline. Two reasons: literal widening, and the ability to add a field later without touching 200 handlers.

```ts
// ❌ Inline object literals widen `success` to boolean when stored in a variable:
const bad = { success: true, data: user };
//    ^ inferred as { success: boolean; data: User } — NOT a discriminated union member.
//      Assigning it to ApiResponse<User> fails: boolean is not assignable to true.

// ✅ Builders pin the literal via the declared return type:
export function ok<T>(data: T, requestId: string): ApiSuccess<T> {
  return { success: true, data, requestId };
}

export function fail(
  code:     ApiErrorCode,
  message:  string,
  requestId: string,
  details?: unknown,
): ApiFailure {
  return { success: false, error: { code, message, ...(details !== undefined && { details }) }, requestId };
}

// A sender that also picks the status code from the error code — one place, always right:
export function send<T>(res: Response<ApiResponse<T>>, body: ApiResponse<T>): void {
  const status = body.success ? 200 : STATUS_FOR_CODE[body.error.code];
  res.status(status).json(body);
}

export function sendCreated<T>(res: Response<ApiResponse<T>>, body: ApiSuccess<T>): void {
  res.status(201).json(body);
}
```

Adding `serverTime` to every success response later is now a one-line change in `ok()` rather than a codemod.

### Concept 5 — Typing `res.json` with `Response<ApiResponse<T>>`

Express's `Response` type takes the response-body type as its first generic parameter. Supplying it turns `res.json` from a wildcard into a contract.

```ts
import type { Request, Response } from "express";

// Response<ResBody, Locals> — we only care about ResBody here.
type ApiRes<T> = Response<ApiResponse<T>>;

// Request<Params, ResBody, ReqBody, Query> — annotate params and body too:
type ApiReq<Params = unknown, Body = unknown, Query = unknown> =
  Request<Params, unknown, Body, Query>;

export async function getUserHandler(
  req: ApiReq<{ userId: string }>,
  res: ApiRes<UserDto>,
): Promise<void> {
  const user = await userRepo.findById(req.params.userId);   // req.params.userId: string ✅
  if (!user) {
    send(res, fail("NOT_FOUND", "User not found", req.requestId));
    return;
  }
  send(res, ok(toUserDto(user), req.requestId));

  // ── Everything below would be a compile error: ───────────────────────────
  // res.json(user);                       // ❌ not an ApiResponse
  // res.json({ success: true });          // ❌ missing `data` and `requestId`
  // res.json(ok(user, req.requestId));    // ❌ User is not assignable to UserDto
}
```

Two caveats worth knowing up front:
- `res.send()` is typed against the same `ResBody`, but `res.end()` is not — it takes strings/buffers and bypasses the check.
- The generic constrains the *argument*, not the *number of calls*. Calling `res.json` twice still compiles (and still throws `ERR_HTTP_HEADERS_SENT` at runtime). Returning `void` from handlers and using `return send(...)` style helps you spot double sends by eye.

### Concept 6 — Paginated responses as a generic

Pagination metadata is part of the payload, not part of the envelope. Nesting it inside `data` keeps `ApiResponse<T>` a two-member union forever.

```ts
export interface Page<T> {
  readonly items:      readonly T[];
  readonly totalCount: number;    // total matching rows, ignoring limit/offset
  readonly limit:      number;
  readonly offset:     number;
  readonly hasMore:    boolean;   // derived: offset + items.length < totalCount
}

/** Convenience alias — this is what a list endpoint returns. */
export type PaginatedResponse<T> = ApiResponse<Page<T>>;

export function page<T>(
  items:      readonly T[],
  totalCount: number,
  limit:      number,
  offset:     number,
): Page<T> {
  return { items, totalCount, limit, offset, hasMore: offset + items.length < totalCount };
}

// Cursor pagination is a different Page shape — a union keeps both honest:
export interface CursorPage<T> {
  readonly items:      readonly T[];
  readonly nextCursor: string | null;   // null ⇒ end of the stream
}

export type CursorResponse<T> = ApiResponse<CursorPage<T>>;

// Usage:
export async function listUsersHandler(
  req: ApiReq<unknown, unknown, { limit?: string; offset?: string }>,
  res: Response<PaginatedResponse<UserDto>>,
): Promise<void> {
  const limit  = Math.min(Number(req.query.limit  ?? 25), 100);
  const offset = Math.max(Number(req.query.offset ?? 0), 0);

  const { rows, total } = await userRepo.list({ limit, offset });
  send(res, ok(page(rows.map(toUserDto), total, limit, offset), req.requestId));
}
```

### Concept 7 — DTO vs domain model: the leak that types prevent

Your domain `User` is what the database and business logic use. Your `UserDto` is what leaves the process. They must be **separate types**, and the only bridge between them is a mapper function.

```ts
// ── Domain model — internal, includes secrets and internal-only fields ──────
export interface User {
  readonly userId:            string;
  readonly email:             string;
  readonly passwordHash:      string;      // 🔒 must never leave the process
  readonly totpSecret:        string | null; // 🔒
  readonly fullName:          string;
  readonly internalRiskScore: number;      // 🔒 internal-only
  readonly role:              "admin" | "member" | "viewer";
  readonly createdAt:         Date;
  readonly deletedAt:         Date | null;
}

// ── DTO — the public wire shape. Note: dates are ISO strings. ───────────────
export interface UserDto {
  readonly userId:    string;
  readonly email:     string;
  readonly fullName:  string;
  readonly role:      "admin" | "member" | "viewer";
  readonly createdAt: string;              // JSON has no Date type
}

// ── The one bridge. Explicit field list — additive DB columns can't sneak out.
export function toUserDto(user: User): UserDto {
  return {
    userId:    user.userId,
    email:     user.email,
    fullName:  user.fullName,
    role:      user.role,
    createdAt: user.createdAt.toISOString(),
  };
}
```

Why not `Omit<User, "passwordHash" | "totpSecret">`? Because `Omit` is a **blocklist**: add a `stripeCustomerId` column tomorrow and it is public by default. An explicit interface is an **allowlist** — new columns are private until someone deliberately adds them to the DTO *and* the mapper. Default-deny beats default-allow every time.

You can still get help from the compiler to keep the DTO honest:

```ts
// Compile-time assertion: every DTO field must exist on the domain model with a
// compatible-ish name. Catches typos and removed columns.
type DtoFieldsExistOnDomain = keyof UserDto extends keyof User ? true : never;
const _dtoCheck: DtoFieldsExistOnDomain = true;   // ❌ errors if UserDto gains a stray field

// Compile-time denylist: assert the DTO can never contain secret keys.
type SecretKeys = "passwordHash" | "totpSecret" | "internalRiskScore";
type NoSecrets<T> = Extract<keyof T, SecretKeys> extends never ? T : never;
type CheckedUserDto = NoSecrets<UserDto>;         // = UserDto if clean, `never` if leaking
const _leakCheck: CheckedUserDto = {} as UserDto; // ❌ errors the moment a secret is added
```

### Concept 8 — Sharing response types between server and client

The types above are pure type-level declarations plus a handful of tiny builders. Put them in a package both sides import.

```ts
// packages/api-contract/src/index.ts — no runtime deps, no express, no db driver.
export type { ApiResponse, ApiSuccess, ApiFailure, ApiErrorBody, ApiErrorCode } from "./response";
export type { Page, PaginatedResponse, CursorPage } from "./pagination";
export type { UserDto, OrderDto, CreateUserRequest } from "./dto";
export { STATUS_FOR_CODE } from "./response";
```

```ts
// apps/web/src/apiClient.ts — the client's ENTIRE error-handling surface:
import type { ApiResponse, ApiErrorCode } from "@acme/api-contract";

export class ApiClientError extends Error {
  constructor(readonly code: ApiErrorCode, message: string, readonly requestId: string) {
    super(message);
    this.name = "ApiClientError";
  }
}

/** One typed fetch for the whole frontend. */
export async function apiFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const res  = await fetch(`/api${path}`, {
    ...init,
    headers: { "content-type": "application/json", ...init?.headers },
  });
  // We trust the server to always emit the envelope — that trust is the contract.
  const body = (await res.json()) as ApiResponse<T>;

  if (!body.success) {
    throw new ApiClientError(body.error.code, body.error.message, body.requestId);
  }
  return body.data;
}

// Call sites are now one line and fully typed:
import type { UserDto, Page } from "@acme/api-contract";

const user  = await apiFetch<UserDto>("/users/u_123");        // user: UserDto
const users = await apiFetch<Page<UserDto>>("/users?limit=25"); // users: Page<UserDto>
```

Note the `as ApiResponse<T>` cast: it is an *assumption*, not a proof. The server's typed `res.json` is what makes the assumption true. If you want proof, run a runtime schema parse on the client too — see `56 — Runtime validation with Zod`-style patterns and `48 — unknown vs any`.

---

## Example 1 — basic

```ts
// ── src/http/response.ts ────────────────────────────────────────────────────

export type ApiErrorCode =
  | "BAD_REQUEST"
  | "UNAUTHENTICATED"
  | "FORBIDDEN"
  | "NOT_FOUND"
  | "CONFLICT"
  | "VALIDATION_FAILED"
  | "INTERNAL";

export interface ApiErrorBody {
  readonly code:     ApiErrorCode;
  readonly message:  string;
  readonly details?: unknown;
}

export type ApiResponse<T> =
  | { readonly success: true;  readonly data: T }
  | { readonly success: false; readonly error: ApiErrorBody };

export const STATUS_FOR_CODE: Record<ApiErrorCode, number> = {
  BAD_REQUEST:       400,
  UNAUTHENTICATED:   401,
  FORBIDDEN:         403,
  NOT_FOUND:         404,
  CONFLICT:          409,
  VALIDATION_FAILED: 422,
  INTERNAL:          500,
};

/** Build a success envelope. */
export function ok<T>(data: T): ApiResponse<T> {
  return { success: true, data };
}

/** Build a failure envelope. `ApiResponse<never>` fits any `ApiResponse<T>` slot. */
export function fail(code: ApiErrorCode, message: string, details?: unknown): ApiResponse<never> {
  return { success: false, error: { code, message, details } };
}

// ── src/dto/user.ts ─────────────────────────────────────────────────────────

/** Internal domain model — has secrets. */
export interface User {
  userId:       string;
  email:        string;
  passwordHash: string;
  fullName:     string;
  createdAt:    Date;
}

/** Public wire shape — no secrets, dates as ISO strings. */
export interface UserDto {
  userId:    string;
  email:     string;
  fullName:  string;
  createdAt: string;
}

export function toUserDto(user: User): UserDto {
  return {
    userId:    user.userId,
    email:     user.email,
    fullName:  user.fullName,
    createdAt: user.createdAt.toISOString(),
  };
}

// ── src/routes/users.ts ─────────────────────────────────────────────────────

import type { Request, Response } from "express";

export async function getUser(
  req: Request<{ userId: string }>,
  res: Response<ApiResponse<UserDto>>,   // ← the contract for this endpoint
): Promise<void> {
  const { userId } = req.params;

  if (!/^u_[a-z0-9]{10}$/.test(userId)) {
    res.status(STATUS_FOR_CODE.BAD_REQUEST).json(fail("BAD_REQUEST", "Malformed userId"));
    return;
  }

  const user = await userRepo.findById(userId);
  if (!user) {
    res.status(STATUS_FOR_CODE.NOT_FOUND).json(fail("NOT_FOUND", "User not found"));
    return;
  }

  res.json(ok(toUserDto(user)));   // ✅ only the DTO can go out
  // res.json(ok(user));           // ❌ Object literal may only specify known properties…
                                   //    'passwordHash' does not exist in type 'UserDto'
}

// ── Consuming it in a test or a client ─────────────────────────────────────

function assertUserOk(body: ApiResponse<UserDto>): UserDto {
  if (!body.success) {
    throw new Error(`Expected success, got ${body.error.code}: ${body.error.message}`);
  }
  return body.data;                // narrowed to UserDto — no casts, no optional chaining
}
```

---

## Example 2 — real world backend use case

```ts
// ══════════════════════════════════════════════════════════════════════════
//  A complete typed response layer: envelope, per-code error details,
//  pagination, DTO mapping, builders, an error middleware, and two routes.
// ══════════════════════════════════════════════════════════════════════════

// ── packages/api-contract/response.ts ──────────────────────────────────────

export type ApiErrorCode =
  | "BAD_REQUEST"
  | "UNAUTHENTICATED"
  | "FORBIDDEN"
  | "NOT_FOUND"
  | "CONFLICT"
  | "VALIDATION_FAILED"
  | "RATE_LIMITED"
  | "INTERNAL"
  | "UPSTREAM_UNAVAILABLE";

export interface FieldIssue {
  readonly field:   string;
  readonly message: string;
}

/** Error body is itself discriminated on `code` so `details` is exact per code. */
export type ApiErrorBody =
  | { readonly code: "VALIDATION_FAILED"; readonly message: string; readonly details: readonly FieldIssue[] }
  | { readonly code: "RATE_LIMITED";      readonly message: string; readonly details: { readonly retryAfterSeconds: number } }
  | { readonly code: "CONFLICT";          readonly message: string; readonly details: { readonly conflictingField: string } }
  | {
      readonly code: Exclude<ApiErrorCode, "VALIDATION_FAILED" | "RATE_LIMITED" | "CONFLICT">;
      readonly message: string;
      readonly details?: undefined;
    };

export interface ApiSuccess<T> {
  readonly success:   true;
  readonly data:      T;
  readonly requestId: string;
}

export interface ApiFailure {
  readonly success:   false;
  readonly error:     ApiErrorBody;
  readonly requestId: string;
}

export type ApiResponse<T> = ApiSuccess<T> | ApiFailure;

export interface Page<T> {
  readonly items:      readonly T[];
  readonly totalCount: number;
  readonly limit:      number;
  readonly offset:     number;
  readonly hasMore:    boolean;
}

export type PaginatedResponse<T> = ApiResponse<Page<T>>;

export const STATUS_FOR_CODE: Record<ApiErrorCode, number> = {
  BAD_REQUEST:          400,
  UNAUTHENTICATED:      401,
  FORBIDDEN:            403,
  NOT_FOUND:            404,
  CONFLICT:             409,
  VALIDATION_FAILED:    422,
  RATE_LIMITED:         429,
  INTERNAL:             500,
  UPSTREAM_UNAVAILABLE: 503,
};

// ── packages/api-contract/dto.ts ───────────────────────────────────────────

export interface UserDto {
  readonly userId:    string;
  readonly email:     string;
  readonly fullName:  string;
  readonly role:      "admin" | "member" | "viewer";
  readonly createdAt: string;
}

export interface OrderDto {
  readonly orderId:      string;
  readonly userId:       string;
  readonly totalCents:   number;
  readonly currency:     "USD" | "EUR" | "GBP";
  readonly status:       "pending" | "paid" | "shipped" | "refunded";
  readonly placedAt:     string;
}

export interface CreateUserRequest {
  readonly email:    string;
  readonly password: string;
  readonly fullName: string;
}

// ── src/http/builders.ts ───────────────────────────────────────────────────

import type { Response } from "express";
import {
  type ApiResponse, type ApiSuccess, type ApiFailure, type ApiErrorBody,
  type ApiErrorCode, type Page, STATUS_FOR_CODE,
} from "@acme/api-contract";

export function ok<T>(data: T, requestId: string): ApiSuccess<T> {
  return { success: true, data, requestId };
}

/**
 * Overloads keep `details` REQUIRED for codes that define it and FORBIDDEN for
 * codes that don't — see `23 — Function overloads`.
 */
export function fail(code: "VALIDATION_FAILED", message: string, requestId: string, details: readonly FieldIssue[]): ApiFailure;
export function fail(code: "RATE_LIMITED",      message: string, requestId: string, details: { retryAfterSeconds: number }): ApiFailure;
export function fail(code: "CONFLICT",          message: string, requestId: string, details: { conflictingField: string }): ApiFailure;
export function fail(code: Exclude<ApiErrorCode, "VALIDATION_FAILED" | "RATE_LIMITED" | "CONFLICT">, message: string, requestId: string): ApiFailure;
export function fail(code: ApiErrorCode, message: string, requestId: string, details?: unknown): ApiFailure {
  return { success: false, error: { code, message, details } as ApiErrorBody, requestId };
}

export function paginate<T>(
  items: readonly T[], totalCount: number, limit: number, offset: number,
): Page<T> {
  return { items, totalCount, limit, offset, hasMore: offset + items.length < totalCount };
}

/** Sends the envelope AND derives the HTTP status from the error code. */
export function send<T>(res: Response<ApiResponse<T>>, body: ApiResponse<T>, successStatus = 200): void {
  const status = body.success ? successStatus : STATUS_FOR_CODE[body.error.code];
  res.status(status).json(body);
}

// ── src/domain/user.ts + mapper ────────────────────────────────────────────

export interface User {
  readonly userId:            string;
  readonly email:             string;
  readonly passwordHash:      string;        // 🔒
  readonly totpSecret:        string | null; // 🔒
  readonly internalRiskScore: number;        // 🔒
  readonly fullName:          string;
  readonly role:              "admin" | "member" | "viewer";
  readonly createdAt:         Date;
  readonly deletedAt:         Date | null;
}

/** Allowlist mapper — new DB columns are private until listed here. */
export function toUserDto(user: User): UserDto {
  return {
    userId:    user.userId,
    email:     user.email,
    fullName:  user.fullName,
    role:      user.role,
    createdAt: user.createdAt.toISOString(),
  };
}

// ── src/http/ApiError.ts — throwable errors that carry a code ───────────────

export class ApiError extends Error {
  private constructor(
    readonly code:    ApiErrorCode,
    message:          string,
    readonly details?: unknown,
  ) {
    super(message);
    this.name = "ApiError";
  }

  static notFound(resource: string): ApiError {
    return new ApiError("NOT_FOUND", `${resource} not found`);
  }

  static conflict(message: string, conflictingField: string): ApiError {
    return new ApiError("CONFLICT", message, { conflictingField });
  }

  static validation(issues: readonly FieldIssue[]): ApiError {
    return new ApiError("VALIDATION_FAILED", "Request validation failed", issues);
  }

  static unauthenticated(message = "Missing or invalid authToken"): ApiError {
    return new ApiError("UNAUTHENTICATED", message);
  }
}

// ── src/http/errorMiddleware.ts — the ONLY place 500s are produced ─────────

import type { NextFunction, Request } from "express";

export function errorMiddleware(
  err: unknown,
  req: Request,
  res: Response<ApiResponse<never>>,
  _next: NextFunction,
): void {
  const requestId = res.locals.requestId as string;

  if (err instanceof ApiError) {
    logger.warn("Handled API error", { requestId, code: err.code, message: err.message });
    res.status(STATUS_FOR_CODE[err.code]).json({
      success: false,
      requestId,
      error: { code: err.code, message: err.message, details: err.details } as ApiErrorBody,
    });
    return;
  }

  // Unknown throwable: log everything internally, expose nothing externally.
  logger.error("Unhandled error", { requestId, err });
  res.status(500).json({
    success: false,
    requestId,
    error: { code: "INTERNAL", message: "An unexpected error occurred" },
  });
}

// ── src/routes/users.ts ────────────────────────────────────────────────────

type ApiReq<P = unknown, B = unknown, Q = unknown> = Request<P, unknown, B, Q>;

export async function createUserHandler(
  req: ApiReq<unknown, CreateUserRequest>,
  res: Response<ApiResponse<UserDto>>,
): Promise<void> {
  const requestBody = req.body;
  const requestId   = res.locals.requestId as string;

  const issues: FieldIssue[] = [];
  if (!requestBody.email?.includes("@"))  issues.push({ field: "email",    message: "must be a valid email" });
  if ((requestBody.password ?? "").length < 12) issues.push({ field: "password", message: "must be at least 12 characters" });
  if (!requestBody.fullName?.trim())      issues.push({ field: "fullName", message: "is required" });

  if (issues.length > 0) {
    send(res, fail("VALIDATION_FAILED", "Request validation failed", requestId, issues));
    return;
  }

  const existing = await userRepo.findByEmail(requestBody.email);
  if (existing) {
    send(res, fail("CONFLICT", "Email already registered", requestId, { conflictingField: "email" }));
    return;
  }

  const user = await userService.create(requestBody);
  send(res, ok(toUserDto(user), requestId), 201);   // 201 Created
}

export async function listUsersHandler(
  req: ApiReq<unknown, unknown, { limit?: string; offset?: string; role?: string }>,
  res: Response<PaginatedResponse<UserDto>>,
): Promise<void> {
  const requestId = res.locals.requestId as string;
  const limit     = Math.min(Math.max(Number(req.query.limit ?? 25) || 25, 1), 100);
  const offset    = Math.max(Number(req.query.offset ?? 0) || 0, 0);

  const { rows, totalCount } = await userRepo.list({ limit, offset, role: req.query.role });

  send(res, ok(paginate(rows.map(toUserDto), totalCount, limit, offset), requestId));
  // ❌ send(res, ok(paginate(rows, totalCount, limit, offset), requestId));
  //    Type 'User[]' is not assignable to 'readonly UserDto[]' — passwordHash blocked.
}

// ── apps/web/src/apiClient.ts — the shared types, reused verbatim ───────────

import type { ApiResponse, ApiErrorCode, Page, UserDto } from "@acme/api-contract";

export class ApiClientError extends Error {
  constructor(readonly code: ApiErrorCode, message: string, readonly requestId: string) {
    super(message);
    this.name = "ApiClientError";
  }
}

async function apiFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const authToken = localStorage.getItem("authToken");
  const res = await fetch(`/api${path}`, {
    ...init,
    headers: {
      "content-type": "application/json",
      ...(authToken ? { authorization: `Bearer ${authToken}` } : {}),
      ...init?.headers,
    },
  });

  const body = (await res.json()) as ApiResponse<T>;
  if (!body.success) throw new ApiClientError(body.error.code, body.error.message, body.requestId);
  return body.data;
}

// Fully typed call sites — no `any`, no shape guessing:
export const listUsers  = (limit = 25, offset = 0) =>
  apiFetch<Page<UserDto>>(`/users?limit=${limit}&offset=${offset}`);

export const getUser    = (userId: string) =>
  apiFetch<UserDto>(`/users/${encodeURIComponent(userId)}`);

export const createUser = (requestBody: CreateUserRequest) =>
  apiFetch<UserDto>("/users", { method: "POST", body: JSON.stringify(requestBody) });
```

---

## Going deeper

### 1 — `ApiResponse<never>` is the reason `fail()` works everywhere

`fail()` returns `ApiResponse<never>`. Because the success branch is `{ success: true; data: never }` and `never` is assignable to everything, `ApiResponse<never>` is assignable to `ApiResponse<UserDto>`, `ApiResponse<Page<OrderDto>>`, anything.

```ts
const f: ApiResponse<never>  = fail("NOT_FOUND", "User not found");
const r: ApiResponse<UserDto> = f;   // ✅ compiles — never flows into any T
```

If you instead typed `fail()` as `ApiResponse<unknown>`, this breaks: `unknown` is assignable *from* everything but *to* nothing. This is the practical payoff of understanding `never` — see `07 — Special types`.

### 2 — Excess property checking only fires on fresh object literals

The `passwordHash` leak is caught because `res.json(ok(user))` passes a *variable*… actually no — and that's the trap.

```ts
declare const user: User;

// ✅ Caught: the argument to ok<UserDto>() is inferred from `user`, so T = User,
//    and ApiSuccess<User> is not assignable to ApiSuccess<UserDto>? — WRONG.
//    Structurally, User HAS every UserDto field, so ApiSuccess<User> IS assignable
//    to ApiSuccess<UserDto> except createdAt (Date vs string). That mismatch is
//    what actually saves you here.

// This is why DTOs should use ISO STRING dates: it makes the domain model
// structurally incompatible with the DTO, so the compiler catches the leak.
```

Be precise about what protects you:

```ts
interface UserDto  { userId: string; email: string; createdAt: string }
interface UserRow  { userId: string; email: string; passwordHash: string; createdAt: string }

declare const row: UserRow;
const bad: UserDto = row;      // ✅ COMPILES — structural typing allows extra properties
                               //    on a variable. passwordHash leaks. No error!

const alsoBad = ok<UserDto>(row);          // ✅ compiles too
res.json(ok<UserDto>(row));                // ✅ compiles — passwordHash goes over the wire

const caught: UserDto = { ...row };        // ❌ excess property check DOES fire on a
                                           //    fresh object literal — 'passwordHash'
                                           //    does not exist in type 'UserDto'
```

**The lesson**: a type annotation alone does *not* strip fields, and `JSON.stringify` serialises what is actually on the object, not what the type says. Structural typing permits extra properties on variables. The only reliable protections are:

1. A **mapper function that constructs a fresh literal** (`toUserDto`), because excess property checking fires on object literals. This is why every example above routes through `toUserDto`.
2. Making the domain type structurally *incompatible* (e.g. `createdAt: Date` in domain vs `string` in DTO), so the assignment errors even without a literal.
3. A runtime `pick()`/schema `.parse()` on the way out for defence in depth.

Do not rely on the type annotation alone. Write the mapper.

### 3 — Branded IDs stop the argument-swap bug

`userId: string` and `orderId: string` are the same type, so `getOrder(userId)` compiles. Brands fix that at zero runtime cost:

```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type UserId  = Brand<string, "UserId">;
export type OrderId = Brand<string, "OrderId">;

export interface UserDto  { readonly userId: UserId;  /* … */ }
export interface OrderDto { readonly orderId: OrderId; readonly userId: UserId; /* … */ }

declare function getOrder(orderId: OrderId): Promise<OrderDto>;
declare const someUserId: UserId;

// getOrder(someUserId);   // ❌ Type 'UserId' is not assignable to 'OrderId'
```

Brands survive `JSON.stringify` (they're phantom — nothing exists at runtime), but they do *not* survive `JSON.parse`: a parsed body is `unknown` and needs a cast or a validator to re-brand. See `11 — Literal types` and `10 — Intersection types`.

### 4 — `readonly` on response types is free and worth it

Marking envelope and DTO fields `readonly` costs nothing at runtime and prevents an entire class of bug where a middleware or a test helper mutates the response body after it was built. Note `readonly T[]` for arrays — a plain `T[]` inside a `readonly` property is still mutable in place.

```ts
interface Page<T> {
  readonly items: readonly T[];   // both the property AND the array are frozen
}

declare const p: Page<UserDto>;
// p.items.push(dto);   // ❌ Property 'push' does not exist on type 'readonly UserDto[]'
```

The DX trap: `readonly T[]` is not assignable to `T[]`, so passing `page.items` to a function expecting `UserDto[]` errors. Type your consumers as `readonly UserDto[]` too — see `35 — Readonly class properties`.

### 5 — `res.json` is typed; `res.send`, `res.end` and `next()` are escape hatches

```ts
const res: Response<ApiResponse<UserDto>> = /* … */;

res.json(ok(dto));            // ✅ checked
res.send(ok(dto));            // ✅ checked — same ResBody generic
res.end(JSON.stringify(raw)); // ⚠️ NOT checked — end() takes string|Buffer
res.write("{}");              // ⚠️ NOT checked
```

Ban `res.end`/`res.write` for JSON bodies with an ESLint `no-restricted-properties` rule. The type system can only guard the doors you route through it.

Also: `Response<T>` does not stop you from calling `res.json` twice, nor from forgetting to respond at all. `noImplicitReturns` plus returning `void` from every handler and always writing `return send(...)` (or `send(...); return;`) makes missing/duplicated sends visually obvious.

### 6 — Envelope vs. bare payload: the honest trade-off

An envelope duplicates information HTTP already carries (status codes, `Retry-After`). Purists prefer bare payloads plus RFC 7807 `application/problem+json` for errors. Both work. What does *not* work is doing neither consistently.

If you choose the envelope, still set correct HTTP statuses — that is what proxies, CDNs, load balancers, and retry libraries read. `success: false` with a 200 status is the worst of both worlds: it defeats every piece of infrastructure between you and the client.

### 7 — Generic inference through the envelope has one sharp edge

```ts
// ❌ TypeScript cannot infer T here — it is only used in the return position:
declare function badFetch<T>(path: string): Promise<T>;
const u = await badFetch("/users/1");   // T infers as `unknown`, and every property access errors

// The call site MUST supply it:
const u2 = await apiFetch<UserDto>("/users/1");   // ✅
```

This is an *unsound-by-design* API: `apiFetch<UserDto>` is a promise from the caller, not a proof. Two ways to make it safer:

```ts
// (a) A route registry maps paths to response types — no per-call generic:
interface RouteMap {
  "GET /users":          Page<UserDto>;
  "GET /users/:userId":  UserDto;
  "POST /users":         UserDto;
  "GET /orders/:orderId": OrderDto;
}

async function typedFetch<K extends keyof RouteMap>(
  route: K, path: string, init?: RequestInit,
): Promise<RouteMap[K]> { /* … */ }

const user = await typedFetch("GET /users/:userId", "/users/u_123");  // user: UserDto ✅

// (b) Pass a runtime parser and infer T from it (Zod-style):
async function parsedFetch<T>(path: string, parse: (raw: unknown) => T): Promise<T> {
  const body = (await fetch(path).then((r) => r.json())) as ApiResponse<unknown>;
  if (!body.success) throw new ApiClientError(body.error.code, body.error.message, body.requestId);
  return parse(body.data);   // T inferred from `parse` — and verified at runtime
}
```

### 8 — Serialisation drift: `Date`, `Map`, `Set`, `bigint`, `undefined`

`JSON.stringify` silently mangles values that TypeScript happily types:

| Value in TS | After `JSON.stringify` → `JSON.parse` |
|---|---|
| `Date` | ISO `string` |
| `undefined` (object property) | property **removed** |
| `Map` / `Set` | `{}` |
| `bigint` | **throws** `TypeError` |
| `NaN`, `Infinity` | `null` |
| class instance | plain object, methods gone |

So `ApiResponse<User>` where `User.createdAt: Date` is a **lie** on the client — it's a string there. This is the strongest practical argument for separate DTO types: DTOs use only JSON-safe types (`string | number | boolean | null | arrays | plain objects`), so the type on the client is true.

You can even enforce it:

```ts
type JsonPrimitive = string | number | boolean | null;
type JsonValue = JsonPrimitive | readonly JsonValue[] | { readonly [k: string]: JsonValue };

/** `T` must be JSON-safe, else this resolves to `never`. */
type JsonSafe<T> = T extends JsonValue ? T : never;

type CheckUserDto = JsonSafe<UserDto>;   // = UserDto ✅ (all strings)
// If someone adds `createdAt: Date` to UserDto, CheckUserDto becomes `never`.
```

### 9 — Keep the error `message` untyped, the `code` typed

Clients must branch on `code`, never on `message`. Typing `message` as `string` (not a literal union) is deliberate: it signals that message text is *not* part of the contract and can be reworded, localised, or enriched freely. Anything a client needs to act on belongs in `code` or `details`.

---

## Common mistakes

### Mistake 1 — Returning the domain model instead of a DTO

```ts
// ❌ WRONG — `user` is the DB row: passwordHash, totpSecret, internalRiskScore all ship.
app.get("/users/:userId", async (req, res) => {
  const user = await userRepo.findById(req.params.userId);
  res.json({ success: true, data: user });
});

// ❌ ALSO WRONG — the annotation does NOT strip fields. This compiles and still leaks,
//    because structural typing permits extra properties on a variable.
const data: UserDto = user;
res.json(ok(data));

// ✅ RIGHT — a mapper that builds a FRESH literal, listing every public field.
export function toUserDto(user: User): UserDto {
  return {
    userId:    user.userId,
    email:     user.email,
    fullName:  user.fullName,
    role:      user.role,
    createdAt: user.createdAt.toISOString(),
  };
}
res.json(ok(toUserDto(user)));
```

### Mistake 2 — Typing the error code as `string`

```ts
// ❌ WRONG — `string` gives clients nothing to branch on and no exhaustiveness.
interface ApiErrorBody {
  code:    string;
  message: string;
}

function handle(error: ApiErrorBody) {
  if (error.code === "NOT_FOUn D") { /* typo — compiles fine, never matches */ }
}

// ✅ RIGHT — a closed union: typos are compile errors, switches can be exhaustive.
type ApiErrorCode = "NOT_FOUND" | "CONFLICT" | "VALIDATION_FAILED" | "INTERNAL";

interface ApiErrorBody {
  code:    ApiErrorCode;
  message: string;
}

function handle(error: ApiErrorBody) {
  if (error.code === "NOT_FOUn D") { /* ❌ Error: this comparison is unintentional */ }
}
```

### Mistake 3 — Using `boolean` instead of a boolean *literal* for the discriminant

```ts
// ❌ WRONG — `success: boolean` on both members: not a discriminated union, no narrowing.
interface ApiResponse<T> {
  success: boolean;
  data?:   T;
  error?:  ApiErrorBody;
}

function unwrap<T>(body: ApiResponse<T>): T {
  if (body.success) {
    return body.data;   // ❌ Type 'T | undefined' is not assignable to 'T'
  }
  throw new Error(body.error!.message);   // `!` everywhere — the smell of a bad model
}

// ✅ RIGHT — literal-typed discriminant, mutually exclusive branches.
type ApiResponse<T> =
  | { success: true;  data: T }
  | { success: false; error: ApiErrorBody };

function unwrap<T>(body: ApiResponse<T>): T {
  if (body.success) return body.data;         // ✅ T — no undefined, no `!`
  throw new Error(body.error.message);        // ✅ error guaranteed present
}
```

### Mistake 4 — Building envelopes inline instead of via builders

```ts
// ❌ WRONG — the object is inferred with `success: boolean` (widened), so it does
//    not match the union when stored or passed through a helper.
function buildResponse(user: UserDto) {
  const body = { success: true, data: user };   // { success: boolean; data: UserDto }
  return body;
}
const r: ApiResponse<UserDto> = buildResponse(dto);
// ❌ Type 'boolean' is not assignable to type 'false'.

// ✅ RIGHT — a builder with a declared return type pins the literal.
function ok<T>(data: T): ApiSuccess<T> {
  return { success: true, data };   // contextually typed against ApiSuccess<T>
}
const r2: ApiResponse<UserDto> = ok(dto);   // ✅

// ✅ ALSO RIGHT if you must inline — `as const` or an annotation:
const body = { success: true as const, data: dto };   // { success: true; data: UserDto }
```

### Mistake 5 — Putting pagination metadata on the envelope

```ts
// ❌ WRONG — now the envelope has optional fields that are meaningless for 90% of
//    endpoints, and every consumer must handle `total: number | undefined`.
type ApiResponse<T> =
  | { success: true;  data: T; total?: number; limit?: number; offset?: number }
  | { success: false; error: ApiErrorBody };

// ✅ RIGHT — pagination is part of the PAYLOAD. The envelope stays a clean 2-member union.
interface Page<T> {
  items:      readonly T[];
  totalCount: number;
  limit:      number;
  offset:     number;
  hasMore:    boolean;
}
type PaginatedResponse<T> = ApiResponse<Page<T>>;
```

### Mistake 6 — `success: false` with HTTP 200

```ts
// ❌ WRONG — every proxy, CDN, retry library and monitoring tool sees a healthy 200.
//    Your error rate dashboard reads 0% during an outage.
res.status(200).json(fail("UPSTREAM_UNAVAILABLE", "Payment gateway down", requestId));

// ✅ RIGHT — derive the status from the code, in one shared place.
res.status(STATUS_FOR_CODE[body.error.code]).json(body);   // 503
```

---

## Practice exercises

### Exercise 1 — easy

Build the response envelope module from scratch.

Define:

1. `ApiErrorCode` — a union of at least these codes: `"BAD_REQUEST"`, `"UNAUTHENTICATED"`, `"FORBIDDEN"`, `"NOT_FOUND"`, `"CONFLICT"`, `"VALIDATION_FAILED"`, `"INTERNAL"`.
2. `ApiErrorBody` — `{ code, message, details? }`, all `readonly`.
3. `ApiResponse<T>` — a two-member discriminated union on `success`, both members carrying a `requestId: string`.
4. `STATUS_FOR_CODE: Record<ApiErrorCode, number>` — the HTTP status for each code.
5. `ok<T>(data, requestId)` and `fail(code, message, requestId, details?)` builders, with return types that make `fail(...)` assignable to `ApiResponse<AnythingAtAll>`.
6. `unwrap<T>(body: ApiResponse<T>): T` — returns `data` on success, throws on failure.
7. `describeFailure(error: ApiErrorBody): string` — an exhaustive `switch` over `error.code` with a `never` check in the `default` branch.

Then prove to yourself that adding a new code to `ApiErrorCode` produces **exactly two** compile errors.

```ts
// Write your code here
```

### Exercise 2 — medium

Build the DTO layer and typed handlers for a small orders API.

Given these domain models (write them out yourself):

- `User` — `userId`, `email`, `passwordHash`, `totpSecret`, `internalRiskScore`, `fullName`, `role: "admin" | "member" | "viewer"`, `createdAt: Date`, `deletedAt: Date | null`
- `Order` — `orderId`, `userId`, `totalCents`, `currency: "USD" | "EUR" | "GBP"`, `status: "pending" | "paid" | "shipped" | "refunded"`, `internalMarginCents`, `placedAt: Date`, `paymentProviderRef: string`

Write:

1. `UserDto` and `OrderDto` — JSON-safe only (no `Date`, no secrets, no internal fields), all `readonly`.
2. `toUserDto` and `toOrderDto` mappers that build fresh object literals.
3. `Page<T>` and a `paginate<T>(items, totalCount, limit, offset)` helper that derives `hasMore`.
4. A `getOrderHandler(req: Request<{ orderId: string }>, res: Response<ApiResponse<OrderDto>>)` that returns `NOT_FOUND` for a missing order, `FORBIDDEN` when `order.userId !== req.userId`, and the DTO otherwise.
5. A `listOrdersHandler` typed as `Response<ApiResponse<Page<OrderDto>>>` that reads `limit`/`offset` from `req.query` (strings!), clamps `limit` to 1–100, and returns a page.
6. A compile-time check `type CheckOrderDtoIsJsonSafe = ...` that resolves to `OrderDto` when the DTO uses only JSON-safe types and `never` otherwise. Verify it flips to `never` if you change `placedAt` to `Date`.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a shared API contract package and a fully typed client on top of it, with no per-call generic guessing.

Requirements:

1. **Error union with per-code details.** Make `ApiErrorBody` itself a discriminated union on `code`, where:
   - `"VALIDATION_FAILED"` requires `details: readonly FieldIssue[]`
   - `"RATE_LIMITED"` requires `details: { retryAfterSeconds: number }`
   - `"CONFLICT"` requires `details: { conflictingField: string }`
   - every other code forbids `details`
   Use `Exclude<...>` for the catch-all member.

2. **Overloaded `fail()`** whose signatures make `details` required exactly for the three codes above and rejected for the rest.

3. **A route registry** mapping route keys to response payload types:
   ```
   "GET /users"            → Page<UserDto>
   "GET /users/:userId"    → UserDto
   "POST /users"           → UserDto
   "GET /orders"           → Page<OrderDto>
   "GET /orders/:orderId"  → OrderDto
   "POST /orders/:orderId/refund" → OrderDto
   ```

4. **`typedFetch<K extends keyof RouteMap>(route: K, path: string, init?): Promise<RouteMap[K]>`** that unwraps the envelope and throws a typed `ApiClientError` carrying `code`, `message` and `requestId`. No call site should ever need to write an explicit type argument.

5. **A request-body registry** too: `RequestBodyMap` mapping the mutating routes to their request body types, and make `typedFetch` require a `body` argument for exactly those routes and forbid it for the rest (hint: conditional types plus a rest-parameter tuple — see `44 — Conditional types`).

6. **Branded IDs.** Add `UserId` and `OrderId` brands and prove that `typedFetch("GET /orders/:orderId", ...)` returning an `OrderDto` cannot have its `orderId` passed to a function expecting a `UserId`.

7. **A retry wrapper** `withRetry` that retries only on `"UPSTREAM_UNAVAILABLE"` and `"RATE_LIMITED"` (honouring `details.retryAfterSeconds`), and never on 4xx codes — with the code list derived from the type, not hand-written.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── The envelope ───────────────────────────────────────────────────────────
type ApiResponse<T> =
  | { success: true;  data: T;             requestId: string }
  | { success: false; error: ApiErrorBody; requestId: string };

// ── Closed error codes + status map ────────────────────────────────────────
type ApiErrorCode = "NOT_FOUND" | "CONFLICT" | "VALIDATION_FAILED" | "INTERNAL";
const STATUS_FOR_CODE: Record<ApiErrorCode, number> = { /* exhaustive by construction */ };

// ── Builders (never construct envelopes inline) ────────────────────────────
const ok   = <T>(data: T, requestId: string): ApiResponse<T>     => ({ success: true, data, requestId });
const fail = (code: ApiErrorCode, message: string, requestId: string): ApiResponse<never> =>
  ({ success: false, error: { code, message }, requestId });

// ── Typed handler signature ────────────────────────────────────────────────
(req: Request<Params, unknown, ReqBody, Query>, res: Response<ApiResponse<UserDto>>) => void

// ── Pagination lives in the PAYLOAD ────────────────────────────────────────
interface Page<T> { items: readonly T[]; totalCount: number; limit: number; offset: number; hasMore: boolean }
type PaginatedResponse<T> = ApiResponse<Page<T>>;

// ── DTO mapping (fresh literal = excess-property protection) ───────────────
const toUserDto = (u: User): UserDto => ({ userId: u.userId, email: u.email, createdAt: u.createdAt.toISOString() });

// ── Client unwrap ──────────────────────────────────────────────────────────
const body = (await res.json()) as ApiResponse<T>;
if (!body.success) throw new ApiClientError(body.error.code, body.error.message, body.requestId);
return body.data;
```

| Decision | Do this | Not this |
|---|---|---|
| Discriminant | `success: true` / `success: false` (literals) | `success: boolean` with optional `data`/`error` |
| Error code | closed union of string literals | `code: string` |
| Building responses | `ok()` / `fail()` builders | inline object literals |
| `fail()` return type | `ApiResponse<never>` | `ApiResponse<unknown>` |
| Wire shape | dedicated `UserDto` (allowlist) | `Omit<User, "passwordHash">` (blocklist) |
| Stripping secrets | mapper returning a fresh literal | a type annotation on a variable |
| Dates in DTOs | `createdAt: string` (ISO) | `createdAt: Date` |
| Pagination | inside `data` as `Page<T>` | optional fields on the envelope |
| HTTP status | derived from `error.code` | always `200` |
| Client types | imported from a shared contract package | hand-written duplicates |
| `res` typing | `Response<ApiResponse<T>>` | bare `Response` |
| Arrays in DTOs | `readonly T[]` | `T[]` |

---

## Connected topics

- **42 — Discriminated unions** — `ApiResponse<T>` is a discriminated union; everything about narrowing, `assertNever`, and exhaustive switches applies directly here.
- **26 — What are generics** and **28 — Generic interfaces** — `ApiResponse<T>`, `Page<T>` and `apiFetch<T>` are the payoff for understanding generics.
- **32 — Utility types** — `Record<ApiErrorCode, number>`, `Exclude<...>`, `Omit<...>` and why an explicit DTO beats `Omit` for security.
- **11 — Literal types** — boolean and string literal discriminants (`success: true`, `code: "NOT_FOUND"`) are the whole mechanism.
- **23 — Function overloads** — overloaded `fail()` signatures that make `details` required for some codes and forbidden for others.
