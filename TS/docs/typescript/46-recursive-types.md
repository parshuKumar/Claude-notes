# 46 — Recursive Types

## What is this?

A **recursive type** is a type that refers to itself, directly or through a cycle of other types. It is the type-level equivalent of a recursive function: the definition mentions its own name, and the compiler expands it lazily, one level at a time, only as far as it needs to answer the question you asked.

```ts
// A JSON value is: a primitive, OR an array of JSON values, OR an object whose
// values are JSON values. The last two clauses mention JsonValue itself.
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };

const requestBody: JsonValue = {
  userId: 42,
  email: "ada@example.com",
  roles: ["admin", "billing"],
  metadata: { source: "signup", experiments: { pricingV2: true } },
  deletedAt: null,
};
// ✅ Arbitrarily deep, and every level is checked.

// const bad: JsonValue = { userId: 42, createdAt: new Date() };
// ❌ Error: Type 'Date' is not assignable to type 'JsonValue'.
```

Two different things are usually called "recursive types", and this doc covers both:

1. **Recursive data shapes** — a comment with replies that are comments, a category with children that are categories, a filesystem node whose entries are filesystem nodes, a JSON value.
2. **Recursive type *operators*** — a generic type that calls itself with a smaller argument to compute a result: `DeepPartial<T>`, `DeepReadonly<T>`, `Paths<T>`, `Join<Parts>`, `Length<Tuple>`. These are the type system's version of recursion as *computation*, and they are where you meet the compiler's depth limits.

## Why does it matter?

Backend data is recursive far more often than people expect:

- **Comment threads.** A comment has replies; each reply has replies. There is no fixed depth.
- **Category / org trees.** A category has subcategories. An employee has direct reports who have direct reports.
- **Filesystem and S3-prefix walks.** A directory contains files and directories.
- **Permission / ACL trees.** A role inherits from roles that inherit from roles.
- **Query filters.** `{ $and: [ { status: "paid" }, { $or: [...] } ] }` — a Mongo-style filter is a recursive grammar.
- **JSON itself.** Every request body, every webhook payload, every JSONB column, every audit-log diff. If you type these as `any` you have unplugged the type system exactly at the boundary where untrusted data enters.
- **Config merging.** `DeepPartial<AppConfig>` is the only honest type for "overrides", and it is recursive by construction.
- **Type-safe path access.** `getIn(config, "server.cors.origins")` — validating that string against a nested type requires a recursive type that walks the object producing dotted paths.

Without recursive types, each of these degrades into `any`, `Record<string, unknown>`, or a hand-written `Level1`/`Level2`/`Level3` ladder that caps out at whatever depth someone guessed. Recursive types give you *unbounded* structural checking from a definition that is often four lines long.

The flip side — and the reason half this doc is about limits — is that recursion at the type level runs inside the compiler, on every keystroke in your editor. A careless recursive type is how a 2-second build becomes a 40-second build, and how you meet the error *"Type instantiation is excessively deep and possibly infinite."*

## The JavaScript way vs the TypeScript way

There is no JavaScript "way" here at all — JS has no types, so recursive data is validated by hand-written recursive *runtime* code, or not at all.

```js
// JavaScript — a comment thread. The shape lives only in your head.

function renderThread(comment, depth = 0) {
  console.log("  ".repeat(depth) + comment.body);
  // Are replies always present? Sometimes null? Sometimes undefined?
  // Sometimes a string id array instead of objects (after a partial fetch)?
  // Nothing tells you. You find out in production.
  for (const reply of comment.replies || []) {
    renderThread(reply, depth + 1);
  }
}

// Hand-written recursive validator, because the shape is unenforced:
function validateComment(node, path = "$") {
  if (typeof node !== "object" || node === null) throw new Error(`${path}: not an object`);
  if (typeof node.commentId !== "string") throw new Error(`${path}.commentId: expected string`);
  if (typeof node.authorId !== "number") throw new Error(`${path}.authorId: expected number`);
  if (typeof node.body !== "string") throw new Error(`${path}.body: expected string`);
  if (node.replies !== undefined) {
    if (!Array.isArray(node.replies)) throw new Error(`${path}.replies: expected array`);
    node.replies.forEach((child, i) => validateComment(child, `${path}.replies[${i}]`));
  }
  return node;
}

// And the "deep merge" everyone writes, typed as nothing:
function deepMerge(defaults, overrides) {
  const out = { ...defaults };
  for (const key of Object.keys(overrides)) {
    const value = overrides[key];
    out[key] =
      value && typeof value === "object" && !Array.isArray(value)
        ? deepMerge(defaults[key] || {}, value)   // typo in `key`? silently creates a new key
        : value;
  }
  return out;
}

deepMerge(config, { server: { prot: 8080 } });  // ✅ "works". Port never changes. No error, ever.
```

Every guarantee is manual, every typo is a runtime discovery, and the "shape" of a comment is documented by whichever function you happen to read first.

```ts
// TypeScript — write the recursive shape ONCE. The compiler walks it for you,
// to any depth, forever.

interface Comment {
  commentId: string;
  authorId:  number;
  body:      string;
  createdAt: Date;
  replies:   Comment[];        // ← the recursion. That is the entire feature.
}

function renderThread(comment: Comment, depth = 0): void {
  console.log("  ".repeat(depth) + comment.body);
  for (const reply of comment.replies) {   // reply: Comment — inferred, at every level
    renderThread(reply, depth + 1);
  }
}

// Deep structural checking, free, at unbounded depth:
const thread: Comment = {
  commentId: "c_1", authorId: 1, body: "ship it", createdAt: new Date(),
  replies: [
    { commentId: "c_2", authorId: 2, body: "+1", createdAt: new Date(), replies: [
      { commentId: "c_3", authorId: 3, body: "same", createdAt: new Date(), replies: [] },
    ]},
  ],
};

// ❌ Six levels down, one wrong field — still caught, at compile time:
// { ...replies: [{ ...replies: [{ ...replies: [{ commentId: 7, ... }] }] }] }
//   Error: Type 'number' is not assignable to type 'string'.

// And the config merge that JS could never check:
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;

interface AppConfig {
  server:   { port: number; host: string; cors: { origins: string[]; credentials: boolean } };
  database: { url: string; poolSize: number };
}

declare function mergeConfig(defaults: AppConfig, overrides: DeepPartial<AppConfig>): AppConfig;

mergeConfig(defaults, { server: { cors: { origins: ["https://app.example.com"] } } });  // ✅
// mergeConfig(defaults, { server: { prot: 8080 } });
// ❌ Error: Object literal may only specify known properties, and 'prot' does not
//    exist in type 'DeepPartial<{ port: number; host: string; cors: {...} }>'.
```

The satisfying part is the asymmetry of effort: one self-reference in a type buys you infinite-depth checking that would cost a hundred lines of hand-written recursive validation in JavaScript — and unlike the validator, it costs nothing at runtime and can never drift from the shape it describes.

---

## Syntax

```ts
// ── 1. Direct recursion in an interface (the common case, always allowed) ────
interface TreeNode {
  nodeId:   string;
  children: TreeNode[];          // self-reference through an array
  parent:   TreeNode | null;     // self-reference through a union
}

// ── 2. Recursion in a type alias — allowed through arrays, tuples, objects ──
type JsonValue =
  | string | number | boolean | null
  | JsonValue[]                       // ✅ through an array
  | { [key: string]: JsonValue };     // ✅ through an object type

// ── 3. NOT allowed: a type alias that immediately refers to itself ──────────
// type Bad = Bad | string;
// ❌ Error: Type alias 'Bad' circularly references itself.
// The reference must be "deferred" — inside an array, tuple, object, or a
// function's parameter/return position — not at the top level of the alias.

// ── 4. Mutual recursion (two types referring to each other) ─────────────────
interface Category  { categoryId: string; products: Product[] }
interface Product   { productId: string;  category: Category }

// ── 5. Recursive GENERIC type (recursion as computation) ────────────────────
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// ── 6. Recursive conditional type over a tuple ──────────────────────────────
type Reverse<Tuple extends readonly unknown[]> =
  Tuple extends readonly [infer Head, ...infer Rest]
    ? [...Reverse<Rest>, Head]        // recurse on the smaller Rest
    : [];                             // base case: empty tuple

// ── 7. Recursive template-literal type over a string ────────────────────────
type SplitPath<S extends string> =
  S extends `${infer Head}.${infer Tail}` ? [Head, ...SplitPath<Tail>] : [S];
type P = SplitPath<"server.cors.origins">;   // ["server", "cors", "origins"]

// ── 8. Tail-recursive form (accumulator passed down; TS optimises this) ─────
type JoinWith<Parts extends readonly string[], Sep extends string, Acc extends string = ""> =
  Parts extends readonly [infer Head extends string, ...infer Rest extends readonly string[]]
    ? JoinWith<Rest, Sep, Acc extends "" ? Head : `${Acc}${Sep}${Head}`>
    : Acc;

// ── 9. A depth limiter — a tuple used as a countdown counter ────────────────
type Prev = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9];   // Prev[3] = 2
type DeepPartialN<T, Depth extends number = 5> =
  [Depth] extends [never] ? T
  : T extends object ? { [K in keyof T]?: DeepPartialN<T[K], Prev[Depth]> }
  : T;

// ── 10. Escape hatch: stop recursing and go loose ───────────────────────────
type Bounded<T, Depth extends number = 4> =
  Depth extends 0 ? unknown : { [K in keyof T]: Bounded<T[K], Prev[Depth]> };
```

---

## How it works — concept by concept

### Concept 1 — Deferred resolution: why `interface` recurses more easily than `type`

This is the single most common stumbling block, and the explanation is mechanical rather than philosophical.

A **type alias** (`type X = ...`) is *eagerly* resolved: when the compiler sees `type X = ...`, it tries to compute the right-hand side to know what `X` *is*. If computing the right-hand side requires knowing what `X` is, you have an infinite loop, and the compiler reports a circularity error.

An **interface** is *lazily* resolved: an interface declaration creates a named type immediately, and its members are only resolved when someone actually asks for them. So an interface can mention itself anywhere.

```ts
// ── Interfaces: self-reference works everywhere, no ceremony ────────────────
interface LinkedNode {
  value: string;
  next:  LinkedNode | null;    // ✅ fine
}

interface Weird {
  self: Weird;                 // ✅ even a direct, non-optional self-reference is fine
}

// ── Type aliases: self-reference must be DEFERRED behind a structure ────────

// ❌ Immediate self-reference at the top level of the alias:
// type BadUnion = BadUnion | string;
//    Error: Type alias 'BadUnion' circularly references itself.

// ❌ Through an intersection at the top level:
// type BadIntersection = BadIntersection & { extra: string };
//    Error: Type alias 'BadIntersection' circularly references itself.

// ✅ Through an object type — deferred, because property types are lazy:
type GoodObject = { value: string; next: GoodObject | null };

// ✅ Through an array — deferred, because element types are lazy:
type GoodArray = string | GoodArray[];

// ✅ Through a tuple:
type GoodTuple = [string, GoodTuple?] | null;

// ✅ Through a function signature:
type Middleware = (requestBody: unknown, next: Middleware) => Promise<void>;

// ✅ Through a generic's type argument, when that generic defers it:
type GoodPromise = string | Promise<GoodPromise>;
type GoodMap = Map<string, GoodMap | number>;
```

The practical rule: **a type alias may refer to itself as long as the reference sits inside an object, array, tuple, or function type.** A union or intersection *of* those is fine (`type JsonValue = string | JsonValue[]` works because `JsonValue[]` is an array); a union *with itself directly* is not.

Where the interface/alias difference really bites is **recursive generics used in the value position of a mapped type**:

```ts
interface DeepConfig {
  server: { port: number; nested: { deeper: { deepest: { value: string } } } };
}

// A recursive alias here is fine, but the compiler expands it eagerly at each
// usage site and its "display" form gets huge:
type DeepPartialAlias<T> = { [K in keyof T]?: DeepPartialAlias<T[K]> };

// Wrapping the recursion in an interface reintroduces a lazy boundary, which
// keeps error messages readable and instantiation depth lower:
type DeepPartialLazy<T> = T extends object ? DeepPartialBox<T> : T;
interface DeepPartialBox<T> { }   // conceptual: see Concept 7 for the real technique

// A more useful practical version of the same idea — hoist the recursive
// self-reference behind a named interface so tsc caches it:
interface JsonObject { [key: string]: JsonValue }
interface JsonArray extends Array<JsonValue> { }
type JsonValue = string | number | boolean | null | JsonObject | JsonArray;
// ↑ This is the canonical JSON type. The two interfaces exist purely to give
//   the compiler named, lazily-resolved boundaries — it produces dramatically
//   nicer error messages than the fully-inline alias version.
```

Compare the error messages — this is the real payoff:

```ts
type InlineJson = string | number | boolean | null | InlineJson[] | { [k: string]: InlineJson };

// declare const a: InlineJson;
// a = new Date();
// Error: Type 'Date' is not assignable to type
//   'string | number | boolean | InlineJson[] | { [k: string]: InlineJson } | null'.
//   ← the whole union is spelled out, every time, at every depth

// declare const b: JsonValue;
// b = new Date();
// Error: Type 'Date' is not assignable to type 'JsonValue'.
//   ← the named interfaces let the compiler keep the short name
```

### Concept 2 — Recursive data shapes: trees, threads, and linked structures

Recursive *data* is the easy half. Once you write the self-reference, the compiler checks arbitrarily deep literals and narrows correctly inside recursive functions.

```ts
// ── A comment thread ────────────────────────────────────────────────────────
interface Comment {
  commentId: string;
  parentId:  string | null;
  authorId:  number;
  body:      string;
  createdAt: Date;
  replies:   Comment[];
}

// Recursive functions over recursive types type-check perfectly:
function countComments(comment: Comment): number {
  return 1 + comment.replies.reduce((sum, reply) => sum + countComments(reply), 0);
}

function flattenThread(comment: Comment, depth = 0): Array<Comment & { depth: number }> {
  return [
    { ...comment, depth },
    ...comment.replies.flatMap((reply) => flattenThread(reply, depth + 1)),
  ];
}

function findComment(root: Comment, commentId: string): Comment | null {
  if (root.commentId === commentId) return root;
  for (const reply of root.replies) {
    const found = findComment(reply, commentId);
    if (found) return found;          // found: Comment (narrowed from Comment | null)
  }
  return null;
}

// ── A generic tree, so the payload varies but the shape does not ────────────
interface TreeNode<T> {
  value:    T;
  children: TreeNode<T>[];           // ✅ generic self-reference is fine
}

type CategoryTree = TreeNode<{ categoryId: string; name: string; productCount: number }>;

// ── Mutual recursion: two types that need each other ────────────────────────
interface Organisation {
  orgId:       string;
  name:        string;
  departments: Department[];
}
interface Department {
  departmentId: string;
  organisation: Organisation;        // back-reference — a cycle between two types
  employees:    Employee[];
}
interface Employee {
  userId:       number;
  manager:      Employee | null;     // self-reference
  directReports: Employee[];
  department:   Department;          // and back into the cycle
}
// Cycles like this are completely fine for the type checker. They are NOT fine
// for JSON.stringify at runtime — see "Going deeper".

// ── A recursive union: the filter grammar every ORM ends up with ────────────
type FilterNode<T> =
  | { op: "and"; clauses: FilterNode<T>[] }
  | { op: "or";  clauses: FilterNode<T>[] }
  | { op: "not"; clause:  FilterNode<T> }
  | { op: "eq";  field: keyof T; value: T[keyof T] }
  | { op: "gt";  field: keyof T; value: T[keyof T] };

interface OrderRow { orderId: string; userId: number; totalCents: number; status: string }

const filter: FilterNode<OrderRow> = {
  op: "and",
  clauses: [
    { op: "eq", field: "status", value: "paid" },
    { op: "or", clauses: [
      { op: "gt", field: "totalCents", value: 10_000 },
      { op: "eq", field: "userId", value: 42 },
    ]},
  ],
};
// ❌ { op: "eq", field: "totl", value: 1 }  → 'totl' is not a keyof OrderRow.
```

### Concept 3 — The JSON value type, done properly

`JsonValue` deserves its own concept because it is the type you will paste into every project, and there are four variants with meaningfully different behaviour.

```ts
// ── Version A: the canonical one. Named interfaces for readable errors. ─────
type JsonPrimitive = string | number | boolean | null;
interface JsonObject { [key: string]: JsonValue }
interface JsonArray extends Array<JsonValue> {}
type JsonValue = JsonPrimitive | JsonObject | JsonArray;

// ── Version B: all-inline. Works, but errors are unreadable at depth. ───────
type JsonInline =
  | string | number | boolean | null
  | JsonInline[]
  | { [key: string]: JsonInline };

// ── Version C: readonly, for values you never mutate (safer for caches) ─────
type ReadonlyJson =
  | JsonPrimitive
  | readonly ReadonlyJson[]
  | { readonly [key: string]: ReadonlyJson };

// ── Version D: "what JSON.stringify actually accepts" — includes undefined
//    in object positions (keys are dropped) but not in arrays (become null).
//    Most codebases do not model this; know that it exists.
type Serialisable =
  | JsonPrimitive
  | Serialisable[]
  | { [key: string]: Serialisable | undefined }
  | { toJSON(): Serialisable };

// ── Using it at the request boundary ───────────────────────────────────────
function parseJsonBody(raw: string): JsonValue {
  return JSON.parse(raw) as JsonValue;   // JSON.parse returns `any`; this is the
                                         // one honest place to assert JsonValue.
}

// The critical property: you cannot use it without narrowing. That is the point.
function readStringField(body: JsonValue, key: string): string | null {
  if (typeof body !== "object" || body === null || Array.isArray(body)) return null;
  const value = body[key];               // JsonValue
  return typeof value === "string" ? value : null;
}

// ── A recursive walker, fully typed, no `any` anywhere ─────────────────────
function redactSecrets(value: JsonValue, secretKeys: ReadonlySet<string>): JsonValue {
  if (value === null || typeof value !== "object") return value;      // primitive
  if (Array.isArray(value)) return value.map((item) => redactSecrets(item, secretKeys));

  const out: JsonObject = {};
  for (const [key, child] of Object.entries(value)) {
    out[key] = secretKeys.has(key) ? "[REDACTED]" : redactSecrets(child, secretKeys);
  }
  return out;
}

const logged = redactSecrets(
  { userId: 42, authToken: "tok_live_abc", nested: { passwordHash: "$2b$..." } },
  new Set(["authToken", "passwordHash"]),
);
// { userId: 42, authToken: "[REDACTED]", nested: { passwordHash: "[REDACTED]" } }
```

Note what `Object.entries(value)` gives you: because `value` was narrowed to `JsonObject` (an index signature of `JsonValue`), `child` is `JsonValue` and the recursion type-checks with no casts.

### Concept 4 — Recursive utility types: DeepPartial and DeepReadonly

Now recursion as *computation*. These two are the workhorses, and both need guards against built-in objects.

```ts
// ── DeepPartial — every property, at every depth, becomes optional ──────────
type DeepPartial<T> =
  T extends (infer Element)[]                        ? DeepPartial<Element>[]
  : T extends ReadonlyArray<infer Element>           ? ReadonlyArray<DeepPartial<Element>>
  : T extends Date | RegExp | Function | Map<any, any> | Set<any> ? T
  : T extends object                                 ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

interface AppConfig {
  server: {
    port: number;
    host: string;
    tls: { enabled: boolean; certPath: string; keyPath: string };
    cors: { origins: string[]; credentials: boolean };
  };
  database: { url: string; poolSize: number; ssl: { rejectUnauthorized: boolean } };
  featureFlags: Record<string, boolean>;
  startedAt: Date;
}

type ConfigOverrides = DeepPartial<AppConfig>;

const overrides: ConfigOverrides = {
  server: { tls: { enabled: true } },       // ✅ two levels of partial
  database: { poolSize: 20 },
};
// ❌ { server: { tls: { enabled: "yes" } } }  → boolean expected
// ❌ { server: { tsl: { enabled: true } } }   → 'tsl' does not exist

// Note: startedAt stays `Date | undefined`, NOT a mangled bag of Date methods,
// because of the built-in guard. Remove that guard and try it — the result is
// `{ toISOString?: ...; getTime?: ... }` and no real Date is assignable.

// ── DeepReadonly — freeze an entire structure at the type level ─────────────
type DeepReadonly<T> =
  T extends (infer Element)[]                        ? ReadonlyArray<DeepReadonly<Element>>
  : T extends ReadonlyArray<infer Element>           ? ReadonlyArray<DeepReadonly<Element>>
  : T extends Date | RegExp | Function               ? T
  : T extends Map<infer K, infer V>                  ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends Set<infer Element>                     ? ReadonlySet<DeepReadonly<Element>>
  : T extends object                                 ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

declare const frozenConfig: DeepReadonly<AppConfig>;
// frozenConfig.server.port = 4000;              // ❌ read-only
// frozenConfig.server.cors.origins.push("x");   // ❌ push does not exist on readonly string[]
const when: Date = frozenConfig.startedAt;        // ✅ still a real Date

// ── DeepRequired — the mirror image, for "after validation" types ───────────
type DeepRequired<T> =
  T extends (infer Element)[]          ? DeepRequired<Element>[]
  : T extends Date | RegExp | Function ? T
  : T extends object                   ? { [K in keyof T]-?: DeepRequired<NonNullable<T[K]>> }
  : T;

// ── DeepNullable — what a JOIN-heavy SQL result actually looks like ─────────
type DeepNullable<T> =
  T extends (infer Element)[]          ? DeepNullable<Element>[] | null
  : T extends Date | RegExp | Function ? T | null
  : T extends object                   ? { [K in keyof T]: DeepNullable<T[K]> } | null
  : T | null;
```

The ordering of the conditional branches matters enormously:

```ts
// ❌ WRONG order — arrays are objects, so this branch swallows them:
type BrokenDeepPartial<T> =
  T extends object ? { [K in keyof T]?: BrokenDeepPartial<T[K]> }
  : T extends (infer E)[] ? BrokenDeepPartial<E>[]   // ← unreachable, dead code
  : T;

type Broken = BrokenDeepPartial<{ origins: string[] }>;
// origins?: { [n: number]?: string; length?: number; push?: ...; ... }
// An array literal is no longer assignable. Always test arrays first.
```

### Concept 5 — Recursive conditional types over tuples and strings

Tuples and template-literal strings are the type system's "lists", and recursion is how you fold over them. This is where recursive types become genuinely computational.

```ts
// ── Tuple recursion: destructure head/rest, recurse on rest ────────────────
type Head<Tuple extends readonly unknown[]> =
  Tuple extends readonly [infer First, ...unknown[]] ? First : never;

type Tail<Tuple extends readonly unknown[]> =
  Tuple extends readonly [unknown, ...infer Rest] ? Rest : [];

type Length<Tuple extends readonly unknown[]> = Tuple["length"];

// Reverse a tuple — classic non-tail recursion:
type Reverse<Tuple extends readonly unknown[]> =
  Tuple extends readonly [infer First, ...infer Rest] ? [...Reverse<Rest>, First] : [];

type R = Reverse<["userId", "email", "createdAt"]>;
// = ["createdAt", "email", "userId"]

// Map every element of a tuple through another type:
type PromisifyAll<Tuple extends readonly unknown[]> =
  Tuple extends readonly [infer First, ...infer Rest]
    ? [Promise<First>, ...PromisifyAll<Rest>]
    : [];
type P = PromisifyAll<[number, string, Date]>;   // [Promise<number>, Promise<string>, Promise<Date>]

// ── String recursion via template literal inference ────────────────────────
type Split<S extends string, Sep extends string> =
  S extends `${infer Before}${Sep}${infer After}`
    ? [Before, ...Split<After, Sep>]
    : [S];

type Segments = Split<"users/:userId/orders/:orderId", "/">;
// = ["users", ":userId", "orders", ":orderId"]

// Extract Express-style route params from a path string:
type RouteParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof RouteParams<Rest>]: string }
    : Path extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

type OrderParams = RouteParams<"/users/:userId/orders/:orderId">;
// = { userId: string; orderId: string }

// ── Building a counter out of a tuple (the standard arithmetic trick) ──────
type BuildTuple<N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N ? Acc : BuildTuple<N, [...Acc, unknown]>;

type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]["length"];

type Sum = Add<3, 4>;    // 7  — yes, the type system did arithmetic
type Zero = BuildTuple<0>;  // []

// Subtract one — the workhorse of depth limiters:
type Decrement<N extends number> =
  BuildTuple<N> extends [unknown, ...infer Rest] ? Rest["length"] : 0;
type Four = Decrement<5>;   // 4
```

### Concept 6 — Tail recursion and why TypeScript cares

Since TypeScript 4.5, the compiler recognises **tail-recursive conditional types** and evaluates them iteratively instead of stacking instantiations. A conditional type is in tail position when the recursive call *is* the whole branch result — nothing wraps it.

```ts
// ── NON tail-recursive: the recursive call is wrapped in a tuple literal ────
type ReverseNonTail<Tuple extends readonly unknown[]> =
  Tuple extends readonly [infer First, ...infer Rest]
    ? [...ReverseNonTail<Rest>, First]        // ← wrapped in [...] → NOT tail position
    : [];
// Depth limit: roughly 50 elements before "excessively deep".

// ── TAIL-recursive: an accumulator carries the partial result down ─────────
type ReverseTail<Tuple extends readonly unknown[], Acc extends unknown[] = []> =
  Tuple extends readonly [infer First, ...infer Rest]
    ? ReverseTail<Rest, [First, ...Acc]>      // ← the call IS the branch result
    : Acc;
// Depth limit: about 1000 iterations. A 20x improvement for one refactor.

type Big = ReverseTail<BuildTuple<200>>;      // ✅ fine
// type BigBroken = ReverseNonTail<BuildTuple<200>>;
// ❌ Error: Type instantiation is excessively deep and possibly infinite.

// ── The same transformation on strings ─────────────────────────────────────

// Non-tail: the recursion is inside a template literal.
type SnakeSlow<S extends string> =
  S extends `${infer Head}${infer Rest}`
    ? Rest extends Uncapitalize<Rest>
      ? `${Uncapitalize<Head>}${SnakeSlow<Rest>}`      // wrapped → not tail
      : `${Uncapitalize<Head>}_${SnakeSlow<Rest>}`
    : S;

// Tail: accumulate the output string as you go.
type SnakeFast<S extends string, Acc extends string = ""> =
  S extends `${infer Head}${infer Rest}`
    ? SnakeFast<
        Rest,
        Head extends Uppercase<Head>
          ? Head extends Lowercase<Head>                 // digits/symbols: both cases equal
            ? `${Acc}${Head}`
            : `${Acc}${Acc extends "" ? "" : "_"}${Lowercase<Head>}`
          : `${Acc}${Head}`
      >
    : Acc;

type Col1 = SnakeFast<"userId">;         // "user_id"
type Col2 = SnakeFast<"createdAtUtc">;   // "created_at_utc"

// ── Tail-recursive join, used everywhere in path builders ──────────────────
type Join<Parts extends readonly string[], Sep extends string, Acc extends string = ""> =
  Parts extends readonly [infer Head extends string, ...infer Rest extends readonly string[]]
    ? Join<Rest, Sep, Acc extends "" ? Head : `${Acc}${Sep}${Head}`>
    : Acc;

type Path = Join<["server", "cors", "origins"], ".">;   // "server.cors.origins"
```

Two rules for keeping tail position:

1. The recursive call must be the *entire* result of the branch — not `[...Call<...>]`, not `` `${Call<...>}` ``, not `Call<...> | X`.
2. Push the "work" into the arguments (the accumulator) instead of doing it to the return value.

### Concept 7 — `Paths<T>`: recursion over keys, the real-world boss fight

`Paths<T>` produces every dotted path into a nested object as a string literal union. It combines mapped types, key remapping, template literals, and recursion — and it is the type behind `config.get("server.cors.origins")` being checked.

```ts
interface AppConfig {
  server:   { port: number; host: string; cors: { origins: string[]; credentials: boolean } };
  database: { url: string; poolSize: number };
  featureFlags: { pricingV2: boolean };
}

// ── Step 1: leaf-only paths, no depth limit yet ────────────────────────────
type Primitive = string | number | boolean | bigint | symbol | null | undefined;

type LeafPaths<T> =
  T extends Primitive | Date | Function | readonly unknown[]
    ? never
    : {
        [K in keyof T & string]: T[K] extends Primitive | Date | readonly unknown[]
          ? K                                          // leaf → the key itself
          : `${K}.${LeafPaths<T[K]>}`;                 // branch → prefix and recurse
      }[keyof T & string];

type ConfigLeaves = LeafPaths<AppConfig>;
// = "server.port" | "server.host" | "server.cors.origins" | "server.cors.credentials"
//   | "database.url" | "database.poolSize" | "featureFlags.pricingV2"

// ── Step 2: include intermediate paths too ─────────────────────────────────
type AllPaths<T> =
  T extends Primitive | Date | Function | readonly unknown[]
    ? never
    : {
        [K in keyof T & string]:
          | K
          | (AllPaths<T[K]> extends infer Sub extends string ? `${K}.${Sub}` : never);
      }[keyof T & string];

type ConfigAll = AllPaths<AppConfig>;
// = "server" | "server.port" | "server.host" | "server.cors" | "server.cors.origins"
//   | "server.cors.credentials" | "database" | "database.url" | ... etc

// ── Step 3: the inverse — given a path, what type lives there? ─────────────
type PathValue<T, Path extends string> =
  Path extends `${infer Head}.${infer Rest}`
    ? Head extends keyof T
      ? PathValue<T[Head], Rest>
      : never
    : Path extends keyof T
      ? T[Path]
      : never;

type OriginsType = PathValue<AppConfig, "server.cors.origins">;   // string[]
type PortType    = PathValue<AppConfig, "server.port">;           // number
type Nope        = PathValue<AppConfig, "server.prot">;           // never

// ── Step 4: put them together into a typed config accessor ─────────────────
declare const config: AppConfig;

function getConfig<Path extends ConfigAll>(path: Path): PathValue<AppConfig, Path> {
  return path.split(".").reduce<any>((acc, key) => acc[key], config);
}

const origins = getConfig("server.cors.origins");   // string[]  ✅
const port    = getConfig("server.port");           // number    ✅
// getConfig("server.prot");
// ❌ Error: Argument of type '"server.prot"' is not assignable to parameter of
//    type '"server" | "server.port" | "server.host" | "server.cors" | ...'
//    ← the autocomplete for this parameter lists every valid path. This is the payoff.

// ── Step 5: bound the depth, because config trees can be deep and wide ─────
type Prev = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

type BoundedPaths<T, Depth extends number = 5> =
  [Depth] extends [never]
    ? never
    : T extends Primitive | Date | Function | readonly unknown[]
      ? never
      : {
          [K in keyof T & string]:
            | K
            | (BoundedPaths<T[K], Prev[Depth]> extends infer Sub extends string
                ? `${K}.${Sub}`
                : never);
        }[keyof T & string];

type SafePaths = BoundedPaths<AppConfig>;   // same result, guaranteed to terminate
```

The `Prev` tuple is the standard depth limiter: `Prev[5]` is `4`, `Prev[1]` is `0`, `Prev[0]` is `never`, and `[Depth] extends [never]` is the terminating check. (The brackets prevent the conditional from distributing over `never`, which would silently return `never` for the whole type — see "Going deeper".)

### Concept 8 — Recursion with a self-referential *value*, not just a type

Recursive types describe recursive runtime structures, and the two must agree. Three patterns show up constantly in backends.

```ts
// ── 1. Building a tree from a flat DB result set ───────────────────────────
interface CategoryRow { categoryId: string; parentId: string | null; name: string }
interface CategoryNode { categoryId: string; name: string; children: CategoryNode[] }

function buildCategoryTree(rows: readonly CategoryRow[]): CategoryNode[] {
  const byId = new Map<string, CategoryNode>();
  for (const row of rows) {
    byId.set(row.categoryId, { categoryId: row.categoryId, name: row.name, children: [] });
  }

  const roots: CategoryNode[] = [];
  for (const row of rows) {
    const node = byId.get(row.categoryId)!;
    if (row.parentId === null) {
      roots.push(node);
    } else {
      byId.get(row.parentId)?.children.push(node);   // ✅ children is CategoryNode[]
    }
  }
  return roots;
}

// ── 2. A recursive type guard that validates untyped input ────────────────
function isCommentTree(value: unknown): value is Comment {
  if (typeof value !== "object" || value === null) return false;
  const candidate = value as Record<string, unknown>;
  return (
    typeof candidate.commentId === "string" &&
    typeof candidate.authorId === "number" &&
    typeof candidate.body === "string" &&
    candidate.createdAt instanceof Date &&
    Array.isArray(candidate.replies) &&
    candidate.replies.every(isCommentTree)          // ← the recursion, at runtime
  );
}

interface Comment {
  commentId: string; authorId: number; body: string; createdAt: Date; replies: Comment[];
}

// ── 3. Lazily-typed recursion for cyclic data (a Zod-style schema) ─────────
// A recursive schema value needs an explicit annotation because inference
// cannot see through the cycle. This is a very common error message.
interface MenuItem { label: string; url: string | null; children: MenuItem[] }

type Validator<T> = { parse(input: unknown): T };

// ❌ Without the annotation, TS reports: "'menuValidator' implicitly has type
//    'any' because it does not have a type annotation and is referenced
//    directly or indirectly in its own initializer."
const menuValidator: Validator<MenuItem> = {
  parse(input: unknown): MenuItem {
    const raw = input as Record<string, unknown>;
    return {
      label: String(raw.label),
      url: raw.url === null ? null : String(raw.url),
      children: (raw.children as unknown[] ?? []).map((child) => menuValidator.parse(child)),
    };
  },
};
```

---

## Example 1 — basic

```ts
// A typed comment thread service: recursive shape, recursive functions,
// recursive derived types — no `any`, unbounded depth.

// ── The recursive domain type ──────────────────────────────────────────────
interface Comment {
  commentId: string;
  postId:    string;
  parentId:  string | null;
  authorId:  number;
  body:      string;
  createdAt: Date;
  isDeleted: boolean;
  replies:   Comment[];          // ← the self-reference
}

// ── The flat row the database actually returns (no recursion here) ─────────
interface CommentRow {
  comment_id: string;
  post_id:    string;
  parent_id:  string | null;
  author_id:  number;
  body:       string;
  created_at: Date;
  is_deleted: boolean;
}

// ── Build the tree from flat rows ──────────────────────────────────────────
function buildCommentTree(rows: readonly CommentRow[]): Comment[] {
  const nodes = new Map<string, Comment>();

  for (const row of rows) {
    nodes.set(row.comment_id, {
      commentId: row.comment_id,
      postId:    row.post_id,
      parentId:  row.parent_id,
      authorId:  row.author_id,
      body:      row.is_deleted ? "[deleted]" : row.body,
      createdAt: row.created_at,
      isDeleted: row.is_deleted,
      replies:   [],
    });
  }

  const roots: Comment[] = [];
  for (const row of rows) {
    const node = nodes.get(row.comment_id)!;
    const parent = row.parent_id === null ? undefined : nodes.get(row.parent_id);
    if (parent) parent.replies.push(node);
    else roots.push(node);        // root, or an orphan whose parent was not fetched
  }
  return roots;
}

// ── Recursive operations — every `reply` is inferred as Comment ────────────

function countReplies(comment: Comment): number {
  return comment.replies.reduce((total, reply) => total + 1 + countReplies(reply), 0);
}

function maxDepth(comment: Comment): number {
  if (comment.replies.length === 0) return 1;
  return 1 + Math.max(...comment.replies.map(maxDepth));
}

function findByAuthor(comment: Comment, authorId: number): Comment[] {
  const here = comment.authorId === authorId ? [comment] : [];
  return [...here, ...comment.replies.flatMap((reply) => findByAuthor(reply, authorId))];
}

// A recursive map that PRESERVES the recursive shape:
function mapComments(comment: Comment, transform: (c: Comment) => Comment): Comment {
  const mapped = transform(comment);
  return { ...mapped, replies: mapped.replies.map((r) => mapComments(r, transform)) };
}

const anonymised = mapComments(thread, (c) => ({ ...c, authorId: 0 }));

// ── A derived recursive type: the API response shape ──────────────────────
// Dates become ISO strings, at EVERY level, recursively.
type Serialised<T> =
  T extends Date                    ? string
  : T extends (infer Element)[]     ? Serialised<Element>[]
  : T extends object                ? { [K in keyof T]: Serialised<T[K]> }
  : T;

type CommentResponse = Serialised<Comment>;
// = { commentId: string; postId: string; parentId: string | null; authorId: number;
//     body: string; createdAt: string; isDeleted: boolean; replies: CommentResponse[] }
//                                                          ↑ recursion survived

function toResponse(comment: Comment): CommentResponse {
  return {
    ...comment,
    createdAt: comment.createdAt.toISOString(),
    replies: comment.replies.map(toResponse),   // ✅ types line up exactly
  };
}

// ── Another derived recursive type: the tree without bodies, for a sidebar ─
type TreeSummary = {
  commentId: string;
  authorId:  number;
  replyCount: number;
  replies:   TreeSummary[];
};

function summarise(comment: Comment): TreeSummary {
  return {
    commentId: comment.commentId,
    authorId:  comment.authorId,
    replyCount: countReplies(comment),
    replies:   comment.replies.map(summarise),
  };
}

declare const thread: Comment;
```

---

## Example 2 — real world backend use case

```ts
// A production-shaped configuration + audit system built entirely on recursive types:
//   PART 1 — a JSON value type at the untrusted boundary, with a recursive differ
//   PART 2 — a typed config service with dotted-path access and deep overrides
//   PART 3 — a recursive permission tree with inheritance resolution

// ════════════════════════════════════════════════════════════════════════════
// PART 1 — JSON at the boundary, and a recursive audit differ
// ════════════════════════════════════════════════════════════════════════════

type JsonPrimitive = string | number | boolean | null;
interface JsonObject { [key: string]: JsonValue }
interface JsonArray extends Array<JsonValue> {}
type JsonValue = JsonPrimitive | JsonObject | JsonArray;

// A change entry for the audit log. `path` is a dotted JSON pointer.
interface FieldChange {
  path: string;
  before: JsonValue | undefined;
  after:  JsonValue | undefined;
}

function isJsonObject(value: JsonValue): value is JsonObject {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

// Recursive structural diff over arbitrary JSON — the core of every audit log.
function diffJson(before: JsonValue, after: JsonValue, path = ""): FieldChange[] {
  // Both plain objects → recurse key by key.
  if (isJsonObject(before) && isJsonObject(after)) {
    const keys = new Set([...Object.keys(before), ...Object.keys(after)]);
    const changes: FieldChange[] = [];
    for (const key of keys) {
      const childPath = path === "" ? key : `${path}.${key}`;
      const beforeChild = before[key];
      const afterChild  = after[key];
      if (beforeChild === undefined || afterChild === undefined) {
        changes.push({ path: childPath, before: beforeChild, after: afterChild });
      } else {
        changes.push(...diffJson(beforeChild, afterChild, childPath));
      }
    }
    return changes;
  }

  // Both arrays → recurse index by index.
  if (Array.isArray(before) && Array.isArray(after)) {
    const changes: FieldChange[] = [];
    const length = Math.max(before.length, after.length);
    for (let index = 0; index < length; index++) {
      const childPath = `${path}[${index}]`;
      const b = before[index];
      const a = after[index];
      if (b === undefined || a === undefined) changes.push({ path: childPath, before: b, after: a });
      else changes.push(...diffJson(b, a, childPath));
    }
    return changes;
  }

  // Leaves, or a type change.
  return before === after ? [] : [{ path: path || "$", before, after }];
}

const auditChanges = diffJson(
  { userId: 42, profile: { displayName: "Ada", tags: ["beta"] }, isAdmin: false },
  { userId: 42, profile: { displayName: "Ada L.", tags: ["beta", "vip"] }, isAdmin: true },
);
// [ { path: "profile.displayName", before: "Ada",   after: "Ada L." },
//   { path: "profile.tags[1]",     before: undefined, after: "vip" },
//   { path: "isAdmin",             before: false,   after: true } ]

// A recursive size guard — reject payload bombs before they reach the DB.
function jsonDepth(value: JsonValue, depth = 0): number {
  if (depth > 64) return depth;                     // hard stop: cycles / attacks
  if (Array.isArray(value)) {
    return value.reduce<number>((max, item) => Math.max(max, jsonDepth(item, depth + 1)), depth);
  }
  if (isJsonObject(value)) {
    return Object.values(value)
      .reduce<number>((max, item) => Math.max(max, jsonDepth(item, depth + 1)), depth);
  }
  return depth;
}

function assertSafePayload(requestBody: JsonValue): void {
  if (jsonDepth(requestBody) > 12) {
    throw new Error("Payload nesting exceeds the maximum supported depth of 12");
  }
}

// ════════════════════════════════════════════════════════════════════════════
// PART 2 — Typed configuration service: dotted paths + deep overrides
// ════════════════════════════════════════════════════════════════════════════

interface AppConfig {
  server: {
    port: number;
    host: string;
    requestTimeoutMs: number;
    cors: { origins: string[]; credentials: boolean; maxAgeSeconds: number };
  };
  database: {
    url: string;
    poolSize: number;
    ssl: { enabled: boolean; rejectUnauthorized: boolean };
  };
  auth: {
    jwtSecret: string;
    accessTokenTtlSeconds: number;
    refresh: { enabled: boolean; ttlDays: number };
  };
  observability: { serviceName: string; sampleRate: number };
}

type Primitive = string | number | boolean | bigint | symbol | null | undefined;
type Prev = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8];

// Every dotted path into the config, bounded at depth 6 so tsc always terminates.
type ConfigPath<T, Depth extends number = 6> =
  [Depth] extends [never]
    ? never
    : T extends Primitive | Date | readonly unknown[] | Function
      ? never
      : {
          [K in keyof T & string]:
            | K
            | (ConfigPath<T[K], Prev[Depth]> extends infer Sub extends string
                ? `${K}.${Sub}`
                : never);
        }[keyof T & string];

// The type stored at a given path.
type ValueAtPath<T, Path extends string> =
  Path extends `${infer Head}.${infer Rest}`
    ? Head extends keyof T ? ValueAtPath<T[Head], Rest> : never
    : Path extends keyof T ? T[Path] : never;

// Deep, safe partial for environment overrides.
type DeepPartial<T> =
  T extends (infer Element)[]                      ? DeepPartial<Element>[]
  : T extends Date | RegExp | Function             ? T
  : T extends object                               ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Deep readonly so nothing mutates config after boot.
type DeepReadonly<T> =
  T extends (infer Element)[]                      ? ReadonlyArray<DeepReadonly<Element>>
  : T extends Date | RegExp | Function             ? T
  : T extends object                               ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

class ConfigService {
  private readonly resolved: DeepReadonly<AppConfig>;

  constructor(defaults: AppConfig, ...layers: Array<DeepPartial<AppConfig>>) {
    const merged = layers.reduce<AppConfig>(
      (acc, layer) => deepMerge(acc, layer),
      structuredClone(defaults),
    );
    this.resolved = merged as DeepReadonly<AppConfig>;
  }

  // Fully typed dotted access: the path is checked, the return type is computed.
  get<Path extends ConfigPath<AppConfig>>(
    path: Path,
  ): DeepReadonly<ValueAtPath<AppConfig, Path>> {
    let current: unknown = this.resolved;
    for (const segment of path.split(".")) {
      current = (current as Record<string, unknown>)[segment];
    }
    return current as DeepReadonly<ValueAtPath<AppConfig, Path>>;
  }
}

// A deep merge whose signature is honest about what it accepts and returns.
function deepMerge<T extends object>(base: T, overrides: DeepPartial<T>): T {
  const out = { ...base };
  for (const key of Object.keys(overrides) as Array<keyof T>) {
    const overrideValue = overrides[key as keyof DeepPartial<T>];
    if (overrideValue === undefined) continue;
    const baseValue = base[key];
    out[key] =
      isPlainObject(baseValue) && isPlainObject(overrideValue)
        ? (deepMerge(baseValue, overrideValue as DeepPartial<typeof baseValue>) as T[keyof T])
        : (overrideValue as T[keyof T]);
  }
  return out;
}

function isPlainObject(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value)
    && !(value instanceof Date) && !(value instanceof RegExp);
}

declare const defaultConfig: AppConfig;

const configService = new ConfigService(
  defaultConfig,
  { server: { port: 8080, cors: { origins: ["https://app.example.com"] } } },
  { database: { ssl: { rejectUnauthorized: false } } },
);

const corsOrigins = configService.get("database.ssl.rejectUnauthorized");  // boolean
const ttl         = configService.get("auth.refresh.ttlDays");             // number
const serverBlock = configService.get("server.cors");
// = DeepReadonly<{ origins: string[]; credentials: boolean; maxAgeSeconds: number }>

// ❌ configService.get("auth.refresh.ttlDayz");
//    Argument of type '"auth.refresh.ttlDayz"' is not assignable to parameter of
//    type '"server" | "server.port" | ... | "auth.refresh.ttlDays" | ...'
// ❌ new ConfigService(defaultConfig, { server: { prot: 1 } });
//    Object literal may only specify known properties; 'prot' does not exist.
// ❌ corsOrigins is readonly at every level:
//    configService.get("server.cors").origins.push("x")  → push does not exist.

// ════════════════════════════════════════════════════════════════════════════
// PART 3 — Recursive permission tree with inheritance
// ════════════════════════════════════════════════════════════════════════════

type Permission =
  | "users:read" | "users:write" | "users:delete"
  | "orders:read" | "orders:write" | "orders:refund"
  | "billing:read" | "billing:write"
  | "admin:*";

// A role node that inherits from other role nodes — recursive by definition.
interface RoleNode {
  roleId:      string;
  displayName: string;
  grants:      Permission[];
  denies:      Permission[];
  inherits:    RoleNode[];      // ← recursion
}

const readOnly: RoleNode = {
  roleId: "read_only", displayName: "Read Only",
  grants: ["users:read", "orders:read"], denies: [], inherits: [],
};

const support: RoleNode = {
  roleId: "support", displayName: "Support Agent",
  grants: ["orders:refund"], denies: [], inherits: [readOnly],
};

const billingAdmin: RoleNode = {
  roleId: "billing_admin", displayName: "Billing Admin",
  grants: ["billing:read", "billing:write"], denies: [], inherits: [support],
};

const restrictedBilling: RoleNode = {
  roleId: "restricted_billing", displayName: "Restricted Billing",
  grants: [], denies: ["orders:refund"], inherits: [billingAdmin],
};

// Resolve the effective permission set: depth-first, with cycle protection.
function resolvePermissions(
  role: RoleNode,
  visited: Set<string> = new Set(),
): Set<Permission> {
  if (visited.has(role.roleId)) return new Set();      // cycle guard — REQUIRED
  visited.add(role.roleId);

  const effective = new Set<Permission>();
  for (const parent of role.inherits) {
    for (const permission of resolvePermissions(parent, visited)) effective.add(permission);
  }
  for (const permission of role.grants) effective.add(permission);
  for (const permission of role.denies) effective.delete(permission);  // deny wins locally
  return effective;
}

const effective = resolvePermissions(restrictedBilling);
// Set { "users:read", "orders:read", "billing:read", "billing:write" }
//   ← "orders:refund" inherited from support, then denied at the leaf.

function canAccess(role: RoleNode, required: Permission): boolean {
  const granted = resolvePermissions(role);
  return granted.has(required) || granted.has("admin:*");
}

// A recursive type that flattens the role tree into a union of role ids —
// useful for typing an `assumeRole` parameter against a known static hierarchy.
type RoleTree = {
  readonly roleId: string;
  readonly inherits: readonly RoleTree[];
};

type RoleIds<T extends RoleTree> =
  | T["roleId"]
  | (T["inherits"][number] extends infer Child
      ? Child extends RoleTree ? RoleIds<Child> : never
      : never);

const roleHierarchy = {
  roleId: "restricted_billing",
  inherits: [
    { roleId: "billing_admin", inherits: [
      { roleId: "support", inherits: [
        { roleId: "read_only", inherits: [] },
      ]},
    ]},
  ],
} as const satisfies RoleTree;

type KnownRoleIds = RoleIds<typeof roleHierarchy>;
// = "restricted_billing" | "billing_admin" | "support" | "read_only"

declare function assumeRole(userId: number, roleId: KnownRoleIds): Promise<void>;
await assumeRole(42, "support");           // ✅
// await assumeRole(42, "suport");         // ❌ not assignable to KnownRoleIds
```

---

## Going deeper

### The instantiation depth limits, precisely

TypeScript has three separate guards, and knowing which one you hit tells you how to fix it.

```ts
// 1. INSTANTIATION DEPTH — nested type instantiations. Limit: 100 (was 50 pre-4.5).
//    Error: "Type instantiation is excessively deep and possibly infinite."
type DeepNest<N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N ? Acc : [DeepNest<N, [...Acc, unknown]>];   // wrapped → not tail
// type Boom = DeepNest<200>;   // ❌ excessively deep

// 2. INSTANTIATION COUNT — total instantiations in one type resolution. Limit: 5,000,000.
//    Same error message; usually means a combinatorial blow-up over a union.

// 3. TAIL-RECURSION ITERATION LIMIT — for recognised tail calls. Limit: 1000.
type Count<N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N ? Acc["length"] : Count<N, [...Acc, unknown]>;  // tail call
type C1 = Count<500>;    // ✅ 500
// type C2 = Count<2000>; // ❌ exceeds the 1000-iteration tail budget

// The practical takeaway: making a type tail-recursive raises your effective
// ceiling from ~50 to ~1000. That is the single highest-leverage refactor.
```

The error is *not* a bug report — it is the compiler refusing to hang. When you see it, the choices are: make the recursion tail-recursive, add a depth limiter, or stop and use a looser type.

### Why `[Depth] extends [never]` and not `Depth extends never`

A subtle trap in every depth limiter.

```ts
type Prev = [never, 0, 1, 2, 3, 4];

// ❌ Naked type parameter → the conditional DISTRIBUTES over the union.
//    `never` is the empty union, so distributing over it produces `never` —
//    the whole type silently collapses to never for EVERY input, not just the base case.
type BadBounded<T, D extends number = 3> =
  D extends never ? T : { [K in keyof T]?: BadBounded<T[K], Prev[D]> };

// ✅ Wrapping both sides in a tuple prevents distribution:
type GoodBounded<T, D extends number = 3> =
  [D] extends [never] ? T : { [K in keyof T]?: GoodBounded<T[K], Prev[D]> };

// Same trick, other common forms:
type IsNever<T> = [T] extends [never] ? true : false;
type IsAny<T>   = 0 extends 1 & T ? true : false;    // only `any` satisfies this
```

Alternatively, count with a tuple length instead of a numeric index — many people find it clearer:

```ts
type DeepPartialD<T, Depth extends unknown[] = [1, 1, 1, 1, 1]> =
  Depth extends [unknown, ...infer Rest]
    ? T extends object ? { [K in keyof T]?: DeepPartialD<T[K], Rest> } : T
    : T;              // ran out of depth → leave the subtree as-is
```

### Recursive types and inference: the "implicitly has type any" error

Inference cannot see through a value that references itself. The type system needs an anchor.

```ts
interface TreeNode { nodeId: string; children: TreeNode[] }

// ❌ 'buildNode' implicitly has return type 'any' because it does not have a
//    return type annotation and is referenced directly or indirectly in one of
//    its return expressions.
// function buildNode(nodeId: string, depth: number) {
//   return { nodeId, children: depth === 0 ? [] : [buildNode(nodeId + ".0", depth - 1)] };
// }

// ✅ Annotate the return type — that breaks the inference cycle:
function buildNode(nodeId: string, depth: number): TreeNode {
  return { nodeId, children: depth === 0 ? [] : [buildNode(`${nodeId}.0`, depth - 1)] };
}

// The same applies to self-referential const objects:
// ❌ const schema = { name: "root", children: [schema] };
// ✅ const schema: TreeNode = { nodeId: "root", children: [] };
//    schema.children.push(schema);   // build the cycle after the annotation exists
```

### Recursive types describe cycles the runtime cannot serialise

The type checker is perfectly happy with `parent`/`children` back-references. `JSON.stringify` is not.

```ts
interface BidirectionalNode {
  nodeId:   string;
  parent:   BidirectionalNode | null;
  children: BidirectionalNode[];
}

const root: BidirectionalNode = { nodeId: "root", parent: null, children: [] };
const child: BidirectionalNode = { nodeId: "child", parent: root, children: [] };
root.children.push(child);

// JSON.stringify(root);   // 💥 TypeError: Converting circular structure to JSON

// The type system cannot warn you about this. Model the WIRE shape separately:
interface SerialisableNode {
  nodeId:   string;
  parentId: string | null;      // an id, not a reference
  children: SerialisableNode[];
}

function toSerialisable(node: BidirectionalNode): SerialisableNode {
  return {
    nodeId: node.nodeId,
    parentId: node.parent?.nodeId ?? null,
    children: node.children.map(toSerialisable),
  };
}
```

A useful rule: **the in-memory type may be cyclic; the DTO type must be acyclic.** Recursive-through-arrays is fine on the wire; recursive-through-back-pointers is not.

### `interface extends Array<Self>` — the trick behind the JSON type

`interface JsonArray extends Array<JsonValue> {}` looks like a hack. It is a deliberate one: it creates a *named* type whose members resolve lazily, which both shortens error messages and reduces the compiler's work relative to the inline `JsonValue[]`.

```ts
// Inline: the alias body is re-expanded at every comparison site.
type Inline = string | Inline[] | { [k: string]: Inline };

// Named: the compiler compares by name first, structurally only when needed.
interface NamedArray extends Array<Named> {}
interface NamedObject { [key: string]: Named }
type Named = string | NamedArray | NamedObject;

// Measurable in large codebases: identical semantics, noticeably faster checking
// and much shorter errors. Prefer the named form for types used across many files.
```

The same idea generalises: **when a recursive alias gets slow or produces unreadable errors, hoist part of it into an interface.** Interfaces are the type system's memoisation boundary.

### Distributive behaviour inside recursive conditionals

Recursive conditional types over unions distribute, which multiplies the work.

```ts
type Wrap<T> = T extends object ? { [K in keyof T]: Wrap<T[K]> } : T;

type Payload = { kind: "a"; data: { x: 1 } } | { kind: "b"; data: { y: 2 } };
type W = Wrap<Payload>;
// = { kind: "a"; data: { x: 1 } } | { kind: "b"; data: { y: 2 } }
//   ← distributed: the recursion ran once per union member. With a 20-member
//     union of deep objects, that is 20 full traversals.

// To evaluate WITHOUT distributing, wrap the checked type in a tuple:
type WrapNoDistribute<T> = [T] extends [object] ? { [K in keyof T]: WrapNoDistribute<T[K]> } : T;
type W2 = WrapNoDistribute<Payload>;
// = { kind: "a" | "b"; data: { x: 1 } | { y: 2 } }   ← different result! usually worse.
```

Usually you *want* distribution here (it preserves discriminated unions). Just be aware it is the main source of combinatorial blow-up when a recursive utility meets a wide union.

### Recursive types over `any` and `unknown`

```ts
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;

type A = DeepPartial<any>;
// = any   — `any extends object` is BOTH branches, and the union collapses to any.
//   A DeepPartial<any> parameter accepts literally anything. Silent hole.

type U = DeepPartial<unknown>;
// = unknown  — `unknown extends object` is false, so it falls to the else branch.

// Guard explicitly when the input might be `any`:
type IsAny<T> = 0 extends 1 & T ? true : false;
type SafeDeepPartial<T> =
  IsAny<T> extends true ? never
  : T extends object ? { [K in keyof T]?: SafeDeepPartial<T[K]> }
  : T;
```

### Recursion terminates on structure, not on values

A recursive type over an *object* terminates because objects have finitely many keys. A recursive type over a *number* or *string* may not.

```ts
// ✅ Terminates: keyof T is finite.
type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> };
// (…unless T is a recursive interface! See the next block.)

// ❌ Does NOT terminate on a recursive interface — infinitely many "levels":
interface Comment { commentId: string; replies: Comment[] }
// type Frozen = DeepReadonly<Comment>;
//   In practice TS handles this because it caches the instantiation
//   DeepReadonly<Comment> and reuses it — but a version that changes the type
//   argument at each step (e.g. adds a wrapper) will blow up.

// This blows up for real, because the argument is different each time:
type WrapForever<T> = { value: WrapForever<{ inner: T }> };
// declare const w: WrapForever<string>;
// w.value.value.value.value;  // ❌ Type instantiation is excessively deep

// ⚠️ String recursion has no natural bound at all:
type RepeatUntil<S extends string, Target extends string> =
  S extends Target ? S : RepeatUntil<`${S}x`, Target>;
// type Never = RepeatUntil<"a", "b">;  // ❌ runs to the iteration limit and errors
```

### When to stop and use a looser type

Recursive types are a tool with a real cost — compile time, editor latency, and error messages people cannot read. Stop and go looser when:

```ts
// ── Signal 1: you hit "excessively deep" and the fix is another depth hack.
//    Two limiters stacked = the type is doing too much.
//    ✅ Loosen the leaves:
type ShallowConfigPatch = {
  [K in keyof AppConfig]?: Partial<AppConfig[K]>;   // 2 levels, not N. Often enough.
};

// ── Signal 2: hover shows a 40-line type. Nobody can debug that.
//    ✅ Name intermediate results:
type ServerOverrides = DeepPartial<AppConfig["server"]>;
type DatabaseOverrides = DeepPartial<AppConfig["database"]>;
interface ConfigOverrides { server?: ServerOverrides; database?: DatabaseOverrides }

// ── Signal 3: the data is genuinely dynamic (user-defined JSON columns,
//    plugin payloads, webhook bodies you do not control).
//    ✅ Use JsonValue and a runtime validator. A recursive TYPE over data you
//       cannot statically know is theatre; a recursive GUARD is real.
interface WebhookEvent {
  eventId:   string;
  eventType: string;
  payload:   JsonValue;              // honest: we do not know the shape
  receivedAt: Date;
}

// ── Signal 4: tsc got slow. Measure before guessing:
//    npx tsc --noEmit --diagnostics
//    npx tsc --noEmit --generateTrace ./trace   (then analyze-trace)
//    Look at "Instantiation count". Recursive utilities are usually the top hit.

// ── Signal 5: the recursive type is only used in ONE place at ONE depth.
//    ✅ Just write the shape out. Three explicit levels beat a clever DeepX<T>
//       that nobody on the team can modify.
interface ExplicitOverrides {
  server?: { port?: number; cors?: { origins?: string[] } };
  database?: { poolSize?: number };
}
```

The honest hierarchy, in order of preference: **exact literal type → bounded recursive type → recursive type with a depth limiter → `JsonValue` plus a runtime validator → `unknown` plus a validator.** `any` is not on the list.

---

## Common mistakes

### Mistake 1 — A type alias that refers to itself immediately

```ts
// ❌ The reference is at the top level of the alias — nothing defers it:
// type Nested = Nested | string;
//    Error: Type alias 'Nested' circularly references itself.

// type Chain = Chain & { requestId: string };
//    Error: Type alias 'Chain' circularly references itself.

// type Deep<T> = Deep<T>;
//    Error: Type alias 'Deep' circularly references itself.

// ✅ Defer the reference behind an object, array, tuple, or function type:
type NestedGood = string | NestedGood[];                  // via array
type ChainGood  = { requestId: string; next: ChainGood | null };  // via object
type Middleware = (requestBody: unknown, next: Middleware) => Promise<void>;  // via function

// ✅ Or use an interface, which is lazily resolved and has no such restriction:
interface ChainNode {
  requestId: string;
  next: ChainNode | null;
}
interface SelfReferential {
  self: SelfReferential;      // ✅ even a direct one
}
```

### Mistake 2 — A deep utility type that mangles `Date`, arrays, and functions

```ts
interface AuditEntry {
  entryId:    string;
  occurredAt: Date;
  tags:       string[];
  onCommit:   () => void;
  metadata:   { source: string; retries: number };
}

// ❌ No guards: Date becomes a bag of optional methods, the array becomes an
//    object with `length?`, and the function loses its call signature.
type BadDeepPartial<T> = T extends object ? { [K in keyof T]?: BadDeepPartial<T[K]> } : T;

type Bad = BadDeepPartial<AuditEntry>;
declare const bad: Bad;
// const when: Date | undefined = bad.occurredAt;
// ❌ Type '{ toISOString?: ...; getTime?: ...; ... }' is not assignable to 'Date | undefined'
// bad.tags?.push("x");        // ❌ 'push' is possibly undefined and wrongly typed
// bad.onCommit?.();           // ❌ call signature dropped by the mapped type

const entry: AuditEntry = {
  entryId: "e1", occurredAt: new Date(), tags: ["billing"],
  onCommit: () => {}, metadata: { source: "api", retries: 0 },
};
// const patch: Bad = { occurredAt: new Date() };
// ❌ Date is not assignable to the mangled shape.

// ✅ Test arrays FIRST, then short-circuit built-ins, then map objects:
type GoodDeepPartial<T> =
  T extends (infer Element)[]                        ? GoodDeepPartial<Element>[]
  : T extends ReadonlyArray<infer Element>           ? ReadonlyArray<GoodDeepPartial<Element>>
  : T extends Date | RegExp | Function | Map<any, any> | Set<any> | Promise<any> ? T
  : T extends object                                 ? { [K in keyof T]?: GoodDeepPartial<T[K]> }
  : T;

type Good = GoodDeepPartial<AuditEntry>;
const patch: Good = { occurredAt: new Date(), metadata: { retries: 3 } };   // ✅
declare const good: Good;
good.onCommit?.();                     // ✅ still callable
good.tags?.map((t) => t.toUpperCase()); // ✅ still a real string[]
```

### Mistake 3 — Non-tail recursion where a tail-recursive form exists

```ts
// ❌ The recursive call is wrapped, so every level stacks an instantiation.
//    Fails around 50 elements.
type BuildPathNonTail<Parts extends readonly string[]> =
  Parts extends readonly [infer Head extends string, ...infer Rest extends readonly string[]]
    ? Rest extends readonly []
      ? Head
      : `${Head}.${BuildPathNonTail<Rest>}`      // ← wrapped in a template literal
    : "";

// type Long = BuildPathNonTail<LongTupleOf60Strings>;
// ❌ Error: Type instantiation is excessively deep and possibly infinite.

// ✅ Carry an accumulator so the recursive call IS the branch result.
//    Now good to ~1000 elements.
type BuildPathTail<
  Parts extends readonly string[],
  Acc extends string = "",
> = Parts extends readonly [infer Head extends string, ...infer Rest extends readonly string[]]
  ? BuildPathTail<Rest, Acc extends "" ? Head : `${Acc}.${Head}`>
  : Acc;

type Short = BuildPathTail<["server", "cors", "origins"]>;   // "server.cors.origins"

// Same fix for tuple building:
// ❌ type FlattenBad<T extends readonly unknown[]> =
//      T extends [infer H, ...infer R]
//        ? H extends readonly unknown[] ? [...FlattenBad<H>, ...FlattenBad<R>] : [H, ...FlattenBad<R>]
//        : [];
// ✅ type FlattenGood<T extends readonly unknown[], Acc extends unknown[] = []> =
//      T extends [infer H, ...infer R]
//        ? FlattenGood<R, H extends readonly unknown[] ? [...Acc, ...H] : [...Acc, H]>
//        : Acc;
```

### Mistake 4 — Recursing without a depth limit over user-shaped data

```ts
interface PluginManifest {
  pluginId: string;
  settings: Record<string, unknown>;   // arbitrary depth, defined by third parties
}

// ❌ Unbounded Paths over a type you do not control:
type UnboundedPaths<T> =
  T extends object
    ? { [K in keyof T & string]: K | `${K}.${UnboundedPaths<T[K]>}` }[keyof T & string]
    : never;
// Applied to a deeply nested plugin config, this either explodes the compile time
// or errors with "excessively deep". And Record<string, unknown> gives you
// `${string}.${string}.${string}...` — an infinitely widening template type.

// ✅ Bound it, and bail out to `string` past the limit:
type Prev = [never, 0, 1, 2, 3, 4];
type BoundedPaths<T, Depth extends number = 4> =
  [Depth] extends [never]
    ? never
    : T extends object
      ? {
          [K in keyof T & string]:
            | K
            | (BoundedPaths<T[K], Prev[Depth]> extends infer Sub extends string
                ? `${K}.${Sub}`
                : never);
        }[keyof T & string]
      : never;

// ✅ Better still for genuinely dynamic data — stop typing it statically:
function getSetting(manifest: PluginManifest, path: string): unknown {
  return path.split(".").reduce<unknown>(
    (acc, key) => (isPlainObject(acc) ? acc[key] : undefined),
    manifest.settings,
  );
}
function isPlainObject(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}
```

### Mistake 5 — Forgetting the runtime cycle guard

```ts
interface RoleNode { roleId: string; grants: string[]; inherits: RoleNode[] }

// ❌ The TYPE permits cycles; this function will recurse forever on one.
function collectGrantsBad(role: RoleNode): string[] {
  return [...role.grants, ...role.inherits.flatMap(collectGrantsBad)];
}

const a: RoleNode = { roleId: "a", grants: ["users:read"], inherits: [] };
const b: RoleNode = { roleId: "b", grants: ["users:write"], inherits: [a] };
a.inherits.push(b);                       // ✅ compiles — a cycle, fully type-safe
// collectGrantsBad(a);                   // 💥 RangeError: Maximum call stack size exceeded

// ✅ Track visited nodes. The compiler will never do this for you.
function collectGrants(role: RoleNode, visited = new Set<string>()): string[] {
  if (visited.has(role.roleId)) return [];
  visited.add(role.roleId);
  return [...role.grants, ...role.inherits.flatMap((r) => collectGrants(r, visited))];
}

collectGrants(a);   // ["users:read", "users:write"]
```

### Mistake 6 — Assuming inference works through the recursion

```ts
interface MenuNode { label: string; children: MenuNode[] }

// ❌ Implicit any: the initialiser references the variable being declared.
// const rootMenu = {
//   label: "root",
//   children: [{ label: "settings", children: [] }, rootMenu],
// };
//    Error: 'rootMenu' implicitly has type 'any' because it does not have a type
//    annotation and is referenced directly or indirectly in its own initializer.

// ✅ Annotate first, wire the cycle afterwards:
const rootMenu: MenuNode = { label: "root", children: [{ label: "settings", children: [] }] };
rootMenu.children.push(rootMenu);

// ❌ Same problem in a recursive function without a return annotation:
// function toTree(rows: CategoryRow[], parentId: string | null) {
//   return rows.filter((r) => r.parentId === parentId)
//              .map((r) => ({ ...r, children: toTree(rows, r.categoryId) }));
// }
//    Error: 'toTree' implicitly has return type 'any'.

// ✅ Annotate the return type explicitly:
interface CategoryRow { categoryId: string; parentId: string | null; name: string }
interface CategoryNode { categoryId: string; name: string; children: CategoryNode[] }

function toTree(rows: readonly CategoryRow[], parentId: string | null): CategoryNode[] {
  return rows
    .filter((row) => row.parentId === parentId)
    .map((row) => ({
      categoryId: row.categoryId,
      name: row.name,
      children: toTree(rows, row.categoryId),
    }));
}
```

### Mistake 7 — Using `Depth extends never` in a limiter (silent `never`)

```ts
type Prev = [never, 0, 1, 2, 3];

// ❌ A naked type parameter distributes; distributing over `never` yields `never`,
//    so the "base case" branch is never taken and the whole type collapses.
type BadLimited<T, D extends number = 3> =
  D extends never ? unknown : { [K in keyof T]?: BadLimited<T[K], Prev[D]> };

interface Config { server: { port: number } }
type BadResult = BadLimited<Config>;
// The recursion still runs off the end and Prev[never] resolves oddly; the base
// case never fires as intended.

// ✅ Wrap in a tuple to switch off distribution:
type GoodLimited<T, D extends number = 3> =
  [D] extends [never] ? unknown : { [K in keyof T]?: GoodLimited<T[K], Prev[D]> };

type GoodResult = GoodLimited<Config>;
// = { server?: { port?: number | undefined } | undefined }   ✅ terminates cleanly
```

---

## Practice exercises

### Exercise 1 — easy

Model a filesystem tree and write the recursive operations over it.

```ts
// 1. Define a recursive type `FsNode` as a discriminated union:
//      - { kind: "file";      name: string; sizeBytes: number; modifiedAt: Date }
//      - { kind: "directory"; name: string; entries: FsNode[] }
//
// 2. Write `totalSize(node: FsNode): number` — the sum of all file sizes in the subtree.
//
// 3. Write `findByName(node: FsNode, name: string): FsNode | null` — depth-first search.
//
// 4. Write `listPaths(node: FsNode, prefix?: string): string[]` — every file path
//    in the tree, slash-joined, e.g. ["src/index.ts", "src/routes/users.ts"].
//
// 5. Write `filterTree(node: FsNode, predicate: (n: FsNode) => boolean): FsNode | null`
//    which keeps a directory if it (or any descendant) satisfies the predicate,
//    and returns null when nothing in the subtree matches.
//
// 6. Define `JsonValue` from scratch (with named interfaces for the object and
//    array cases) and write `toJson(node: FsNode): JsonValue` with no `any` and
//    no type assertions.
//
// Every function must narrow on `node.kind` — no casts allowed.
```

```ts
// Write your code here
```

### Exercise 2 — medium

Build the recursive utility types a real config system needs, from scratch, without copying the ones above.

```ts
interface ServiceConfig {
  http: {
    port: number;
    host: string;
    tls: { enabled: boolean; certPath: string; keyPath: string; ciphers: string[] };
  };
  queue: {
    brokerUrl: string;
    concurrency: number;
    retry: { maxAttempts: number; backoffMs: number; jitter: boolean };
  };
  telemetry: { serviceName: string; sampleRate: number; exportedAt: Date };
}

// Implement all of the following WITHOUT using Partial, Readonly, or Required:
//
// 1. DeepPartial<T>  — arrays handled before objects; Date/RegExp/Function preserved.
// 2. DeepReadonly<T> — arrays become ReadonlyArray; Date preserved; verify that
//                      `config.http.tls.ciphers.push("x")` is a compile error.
// 3. DeepRequired<T> — removes `?` and `undefined` at every level.
// 4. DeepNonNullable<T> — strips `null` from every leaf, recursively.
//
// 5. Leaves<T> — a union of the dotted paths to PRIMITIVE leaves only:
//      "http.port" | "http.host" | "http.tls.enabled" | ... | "telemetry.sampleRate"
//    (arrays and Date count as leaves, not as branches).
//
// 6. ValueAt<T, Path> — given a Leaves<T> path, the type stored there.
//
// 7. A `deepFreeze<T>(value: T): DeepReadonly<T>` function whose runtime behaviour
//    (Object.freeze applied recursively) matches its type, including skipping Dates.
//
// 8. A `set<T, Path extends Leaves<T>>(config: T, path: Path, value: ValueAt<T, Path>): T`
//    function that returns a new object with that one leaf replaced, without mutating.
//
// Then demonstrate: a valid set, a wrong-value-type set (compile error), and a
// misspelled path (compile error).
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a type-safe recursive query filter DSL with a compiler for it.

```ts
interface OrderRow {
  orderId:     string;
  userId:      number;
  totalCents:  number;
  status:      "pending" | "paid" | "shipped" | "cancelled";
  placedAt:    Date;
  shippedAt:   Date | null;
  couponCode:  string | null;
  lineItems:   Array<{ sku: string; quantity: number; unitCents: number }>;
}

// Build ALL of the following:
//
// 1. Comparator<V> — the operator object allowed for a column of type V:
//      number | Date  -> { $eq?; $ne?; $gt?; $gte?; $lt?; $lte?; $in? }
//      string         -> { $eq?; $ne?; $like?; $in? }
//      boolean        -> { $eq?; $ne? }
//      T | null       -> the above for T, plus { $isNull?: boolean }
//
// 2. Filter<T> — a RECURSIVE type:
//      | { $and: Filter<T>[] }
//      | { $or:  Filter<T>[] }
//      | { $not: Filter<T> }
//      | { [K in keyof T]?: T[K] | Comparator<T[K]> }     (a leaf condition)
//    Nesting must be unbounded and every field name / operator must be checked.
//
// 3. NestedPaths<T> — dotted paths that also descend into array element types,
//    so "lineItems.sku" and "lineItems.quantity" are valid paths. Bound the
//    recursion at depth 4 with a Prev-tuple limiter and explain (in a comment)
//    which of the three TS depth limits you are protecting against.
//
// 4. compileFilter<T>(filter: Filter<T>): { sql: string; params: unknown[] }
//    — a recursive compiler that walks the filter tree and emits parameterised
//    SQL: `(status = $1 AND (total_cents > $2 OR user_id = $3))`.
//    Column names must be snake_cased by a RECURSIVE template-literal type
//    `SnakeCase<S>` that you write yourself, in TAIL-RECURSIVE form.
//
// 5. A `validateFilter(filter: unknown): filter is Filter<OrderRow>` recursive
//    type guard, with a depth counter that throws past 16 levels of nesting
//    (payload-bomb protection).
//
// 6. Demonstrate:
//    - a three-level nested $and/$or/$not filter that compiles
//    - `{ status: { $gt: "paid" } }` is a compile error (no $gt on string columns)
//    - `{ shippedAt: { $isNull: true } }` compiles (nullable column)
//    - `{ placedAt: { $isNull: true } }` is a compile error (not nullable)
//    - `{ totl: 100 }` is a compile error (unknown column)
//
// 7. Finally: take your NestedPaths<T> and deliberately raise the depth limit
//    until you hit "Type instantiation is excessively deep and possibly infinite".
//    Record the depth at which it happens in a comment, then refactor whichever
//    part you can into tail-recursive form and record the new limit.
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Recursive data shapes ───────────────────────────────────────────────────
interface TreeNode { nodeId: string; children: TreeNode[] }        // interface: always OK
type JsonValue = string | number | boolean | null                   // alias: defer via
               | JsonValue[] | { [k: string]: JsonValue };          //   array/object/fn

// The canonical JSON type (named interfaces → short errors, faster checking):
type JsonPrimitive = string | number | boolean | null;
interface JsonObject { [key: string]: JsonValue }
interface JsonArray extends Array<JsonValue> {}
type JsonValue2 = JsonPrimitive | JsonObject | JsonArray;

// ── What defers a self-reference in a type alias ────────────────────────────
// ✅ inside [] · inside {} · inside a tuple · in a function param/return
// ❌ `type X = X | Y`  ·  `type X = X & Y`  ·  `type X = X`

// ── Deep utility types (ALWAYS test arrays first, guard built-ins) ──────────
type DeepPartial<T> =
  T extends (infer E)[]                ? DeepPartial<E>[]
  : T extends Date | RegExp | Function ? T
  : T extends object                   ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

type DeepReadonly<T> =
  T extends (infer E)[]                ? ReadonlyArray<DeepReadonly<E>>
  : T extends Date | RegExp | Function ? T
  : T extends object                   ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

// ── Paths and path values ───────────────────────────────────────────────────
type Paths<T> = T extends object
  ? { [K in keyof T & string]: K | `${K}.${Paths<T[K]> & string}` }[keyof T & string]
  : never;

type PathValue<T, P extends string> =
  P extends `${infer Head}.${infer Rest}`
    ? Head extends keyof T ? PathValue<T[Head], Rest> : never
    : P extends keyof T ? T[P] : never;

// ── Depth limiter ───────────────────────────────────────────────────────────
type Prev = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
type Bounded<T, D extends number = 5> =
  [D] extends [never] ? T                       // ← tuple-wrap, NOT `D extends never`
  : T extends object ? { [K in keyof T]?: Bounded<T[K], Prev[D]> }
  : T;

// ── Tail recursion (limit ~1000 vs ~50) ─────────────────────────────────────
type Join<P extends readonly string[], S extends string, Acc extends string = ""> =
  P extends readonly [infer H extends string, ...infer R extends readonly string[]]
    ? Join<R, S, Acc extends "" ? H : `${Acc}${S}${H}`>    // call IS the branch → tail
    : Acc;

// ── Tuple/string recursion primitives ───────────────────────────────────────
type Split<S extends string, Sep extends string> =
  S extends `${infer A}${Sep}${infer B}` ? [A, ...Split<B, Sep>] : [S];
type Reverse<T extends readonly unknown[], Acc extends unknown[] = []> =
  T extends readonly [infer H, ...infer R] ? Reverse<R, [H, ...Acc]> : Acc;
type BuildTuple<N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N ? Acc : BuildTuple<N, [...Acc, unknown]>;

// ── Useful guards ───────────────────────────────────────────────────────────
type IsNever<T> = [T] extends [never] ? true : false;
type IsAny<T>   = 0 extends 1 & T ? true : false;
```

| Construct / situation | Behaviour |
|---|---|
| `interface X { self: X }` | ✅ Always legal — interfaces resolve lazily |
| `type X = X \| Y` | ❌ "Type alias circularly references itself" |
| `type X = X[] \| string` | ✅ Reference deferred behind an array |
| `type X = { next: X }` | ✅ Deferred behind an object type |
| `type X = (next: X) => void` | ✅ Deferred behind a function signature |
| `interface A extends Array<B>` | Named lazy boundary — shorter errors, faster checks |
| Non-tail recursive conditional | ~50–100 instantiation depth before it errors |
| Tail-recursive conditional | ~1000 iterations — call must be the whole branch |
| Instantiation count ceiling | 5,000,000 across one resolution |
| `[D] extends [never]` | Correct base-case test — prevents distribution |
| `D extends never` | ❌ Distributes over the empty union → silent `never` |
| `T extends object` before arrays | ❌ Arrays are objects — the array branch is dead code |
| Missing `Date \| RegExp \| Function` guard | Built-ins get mapped into useless property bags |
| `DeepPartial<any>` | `any` — a silent hole; guard with `IsAny<T>` |
| Cyclic value + `JSON.stringify` | 💥 Runtime `TypeError` the type system cannot see |
| Recursive fn without return annotation | ❌ "implicitly has return type 'any'" |
| Recursive const without annotation | ❌ "referenced directly or indirectly in its own initializer" |
| Recursion over `Record<string, unknown>` | Produces infinitely widening `${string}.${string}` paths — bound it |

**Escalation ladder when a recursive type gets painful:** exact literal type → bounded recursive type → depth-limited recursive type → `JsonValue` + runtime validator → `unknown` + runtime validator. Never `any`.

---

## Connected topics

- **43 — Mapped types** — the `{ [K in keyof T]: ... }` loop is the body of nearly every recursive utility type; `DeepPartial` and `DeepReadonly` are mapped types that call themselves.
- **45 — Template literal types** — string recursion (`Split`, `Join`, `SnakeCase`, `Paths`) is built entirely on template-literal inference, and is where tail recursion pays off most.
- **47 — keyof and typeof operators** — `keyof T` bounds every object recursion, and `typeof value` plus `as const` is how you feed a real runtime tree into a recursive type.
- **42 — Discriminated unions** — recursive unions (`FsNode`, `FilterNode`, JSON) are discriminated unions whose variants contain themselves; narrowing on the tag is what makes recursive functions type-check.
- **12 — Type aliases** — explains the alias-vs-interface distinction that decides whether a self-reference is legal.
- **15 — type vs interface** — the deferred-resolution difference in this doc is the most consequential practical reason to reach for `interface`.
- **41 — Type guards** — a recursive type needs a recursive `value is T` guard at the untrusted boundary; the type alone proves nothing about runtime data.
- **58 — Typing API responses** — `JsonValue` at the request boundary, and recursive `Serialised<T>` for turning `Date` into ISO strings at every level.
- **65 — Readonly and immutability patterns** — `DeepReadonly` plus a matching runtime `deepFreeze` is the standard immutable-config pattern.
- **32 — Utility types** — `Partial`, `Readonly`, `Required` are the one-level versions of the deep variants built here.
