# 72 — ESLint with TypeScript

## What is this?

**ESLint with TypeScript** means running ESLint over `.ts` files using `typescript-eslint` — a package that supplies two things ESLint does not have on its own:

1. A **parser** (`@typescript-eslint/parser`) that turns TypeScript source into an AST ESLint can walk. Stock ESLint's parser (`espree`) chokes on `interface`, `enum`, `satisfies`, generics, decorators, and type annotations.
2. A **rule plugin** (`@typescript-eslint/eslint-plugin`) containing ~130 rules that only make sense in a typed language — and, crucially, a subset that can **ask the type checker questions** while linting.

```
  src/services/orderService.ts
            │
            │  @typescript-eslint/parser
            ▼
  TSESTree AST  ──────────────┐
            │                 │  (optional) ts.Program / projectService
            ▼                 ▼
     ESLint rule engine ← TypeScript type checker
            │
            ▼
  src/services/orderService.ts:88:5
    error  Promises must be awaited  @typescript-eslint/no-floating-promises
```

That second arrow — the one into the type checker — is the whole reason this topic is worth a document. Without it, you get a slightly TypeScript-aware version of the linter you already ran on JavaScript. With it, you get rules that can say *"this expression is a `Promise<void>` and nobody is handling it"* — which no amount of syntax analysis could ever determine.

Two distinct products are being configured here, and confusing them causes most of the pain:

| Thing | Job | Fails the build on |
|---|---|---|
| `tsc` | Type checking + emit | Type errors |
| ESLint + typescript-eslint | Code-quality rules | Rule violations |

They overlap in exactly zero rules. That is not an accident, and it is the first thing to internalise.

---

## Why does it matter?

Because `tsc --noEmit` passing tells you almost nothing about whether your Node service is correct.

Here is real, fully type-correct TypeScript that `tsc` compiles without a murmur, and every line is a production incident waiting to happen:

```ts
// Every one of these type-checks cleanly under "strict": true.

async function handleCheckout(orderId: string): Promise<void> {
  chargeCustomer(orderId);           // ⚠️ returns Promise<void>. Not awaited. If it rejects:
                                     //    unhandled rejection → in Node 15+ the process CRASHES.
                                     //    tsc: silent. no-floating-promises: error.

  await auditLog.write({ orderId }); // fine
}

// ⚠️ Express never awaits your handler. If this rejects, the request hangs until
//    the client times out, and the error never reaches your error middleware.
app.get("/orders/:orderId", async (req, res) => {
  const order = await loadOrder(req.params.orderId);
  res.json(order);
});                                   // tsc: silent. no-misused-promises: error.

// ⚠️ `deletedAt` is `Date | null`. An empty string, 0, and null are all falsy —
//    but here the author meant "has a value". It works... until the type changes
//    to `Date | null | ""`. tsc: silent. strict-boolean-expressions: error.
if (user.deletedAt) { /* ... */ }

// ⚠️ orderId is `string`, never nullish. This `?? "unknown"` is dead code —
//    a leftover from before the type was tightened. It hides the fact that
//    the real nullable field is somewhere else. tsc: silent.
//    no-unnecessary-condition: error.
const label = orderId ?? "unknown";

// ⚠️ Marked async, contains no await. Callers now get a Promise for no reason,
//    and the "I'm doing IO" signal is a lie. tsc: silent. require-await: error.
async function formatOrderNumber(orderId: string): Promise<string> {
  return orderId.toUpperCase();
}
```

The value proposition, concretely, for backend work:

- **`no-floating-promises` alone justifies the setup.** In Node ≥ 15, an unhandled promise rejection terminates the process by default. A single un-awaited `sendWebhook(...)` in a hot path is a crash loop.
- **`no-misused-promises` catches the async-Express-handler class of bug**, which is otherwise invisible until a request hangs in production.
- **`consistent-type-imports` prevents runtime imports of type-only modules** — the thing that makes your bundle 3× bigger, or makes a circular import blow up at runtime even though the types were fine.
- **Consistency across a team** — one config, one CI check, zero code-review arguments about `interface` vs `type` or `Array<T>` vs `T[]`.
- **`tsc` is a compiler, not a style or safety tool.** It will never warn you about `any` leaking through your codebase, because `any` is a legal type.

---

## The JavaScript way vs the TypeScript way

```js
// ── The JavaScript way — ESLint on plain Node ────────────────────────────────
// npm i -D eslint
//
// eslint.config.js
import js from "@eslint/js";

export default [
  js.configs.recommended,
  {
    rules: {
      "no-unused-vars": "error",
      "no-undef": "error",       // ← catches typos in identifiers. This is ESLint's
                                 //   substitute for a type system, and it's weak.
      eqeqeq: "error"
    }
  }
];

// What this can see: the syntax tree. Nothing else.
//
// It knows `chargeCustomer(orderId)` is a call expression.
// It does NOT know that call returns a Promise.
// It CANNOT know, because knowing requires resolving imports, reading .d.ts
// files, instantiating generics, and running inference. That's a compiler's job.
//
// So the JS-era rulebook is limited to: unused vars, undefined vars, style,
// and syntactic smells (== vs ===, var vs let, unreachable code).
```

Now point that config at a `.ts` file:

```js
// ── TypeScript, unconfigured — it doesn't even parse ─────────────────────────
//
// $ npx eslint src/services/orderService.ts
//
// src/services/orderService.ts
//   3:24  error  Parsing error: Unexpected token :
//
// Because espree is a JavaScript parser. `function loadOrder(orderId: string)`
// is a syntax error to it. You get exactly one error per file and zero rules run.
//
// And a second, subtler failure: `no-undef` is now actively HARMFUL.
// TypeScript already reports undefined identifiers, far more accurately, and
// no-undef doesn't understand `declare`, ambient types, or global augmentation —
// so it produces false positives on perfectly valid code.
```

```ts
// ── The TypeScript way — typescript-eslint ───────────────────────────────────
// npm i -D eslint typescript typescript-eslint
//
// eslint.config.js
import js from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  js.configs.recommended,
  tseslint.configs.strictTypeChecked,      // ← rules that CONSULT THE TYPE CHECKER
  {
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname }
    }
  }
);

// Now the linter has the same knowledge tsc has. It can answer:
//
//   "Is the type of this expression assignable to Promise<unknown>?"      → no-floating-promises
//   "Is this condition's type always truthy?"                             → no-unnecessary-condition
//   "Is this comparison between two types with no overlap?"               → no-unnecessary-condition
//   "Does this async function contain an await?"                          → require-await
//   "Is this import used only in type position?"                          → consistent-type-imports
//   "Is the awaited value already non-Promise?"                           → await-thenable
//
// None of these are expressible in a syntax-only linter. All of them are
// bugs that ship to production in real Node services.
```

The revelation: **in JavaScript, ESLint was a partial substitute for the type checking you didn't have. In TypeScript, ESLint becomes a consumer of the type checking you do have** — and that flips it from a style tool into a correctness tool. `tsc` answers "are the types consistent?"; ESLint answers "given the types, is this code sensible?"

---

## Syntax

```bash
# ── Installation ─────────────────────────────────────────────────────────────
npm i -D eslint typescript typescript-eslint

# `typescript-eslint` (no @ scope) is the v8+ meta-package. It re-exports:
#   - @typescript-eslint/parser
#   - @typescript-eslint/eslint-plugin
#   - the `config()` helper and all shared configs
# You almost never need to install the two scoped packages directly any more.

npx eslint .                    # lint the whole project (flat config finds files itself)
npx eslint . --fix              # apply auto-fixable rules
npx eslint . --max-warnings 0   # treat any warning as a failure — the CI flag
npx eslint --inspect-config     # opens a UI showing which config applies to which file
```

```js
// ── eslint.config.js — the minimum viable flat config ────────────────────────
// @ts-check
import js from "@eslint/js";
import tseslint from "typescript-eslint";

// tseslint.config() is a typed helper around a plain array. It gives you
// autocomplete + type errors in the config itself. You could export a raw
// array instead; the helper is strictly nicer.
export default tseslint.config(
  // 1. Global ignores. A config object with ONLY `ignores` applies globally.
  //    Replaces .eslintignore, which flat config does not read.
  { ignores: ["dist/**", "coverage/**", "node_modules/**", "*.config.js"] },

  // 2. Base JavaScript rules.
  js.configs.recommended,

  // 3. TypeScript rules — pick ONE tier (see Concept 4).
  ...tseslint.configs.recommendedTypeChecked,

  // 4. Your own layer: parser options, rule overrides, per-file settings.
  {
    files: ["**/*.ts"],
    languageOptions: {
      parserOptions: {
        projectService: true,                  // type-aware linting, auto tsconfig discovery
        tsconfigRootDir: import.meta.dirname   // anchor for resolving that tsconfig
      }
    },
    rules: {
      "@typescript-eslint/no-floating-promises": "error"
    }
  }
);
```

```jsonc
// ── package.json — the scripts you actually run ──────────────────────────────
{
  "type": "module",                     // required if eslint.config.js uses `import`
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:ci": "eslint . --max-warnings 0",
    "typecheck": "tsc --noEmit",        // SEPARATE from lint. Both must run in CI.
    "check": "npm run typecheck && npm run lint:ci"
  }
}
```

```ts
// ── Rule severity values ─────────────────────────────────────────────────────
// "off"   | 0   rule disabled
// "warn"  | 1   reported, exit code still 0  (unless --max-warnings 0)
// "error" | 2   reported, exit code 1

// With options — always [severity, options]:
"@typescript-eslint/no-unused-vars": ["error", {
  argsIgnorePattern: "^_",
  varsIgnorePattern: "^_",
  caughtErrorsIgnorePattern: "^_"
}]
```

```ts
// ── Inline disable comments ──────────────────────────────────────────────────
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- 3rd-party lib is untyped
const legacyClient: any = require("old-sdk");

const x = compute(); // eslint-disable-line @typescript-eslint/no-unsafe-assignment

/* eslint-disable @typescript-eslint/no-unsafe-member-access */
// ...block of code...
/* eslint-enable @typescript-eslint/no-unsafe-member-access */

/* eslint-disable */          // ⚠️ whole file, all rules. Almost never justified.
```

---

## How it works — concept by concept

### Concept 1 — Why `tsc` is not a linter (and never will be)

This is the most common misconception among people coming from JavaScript, where ESLint *was* the safety net. The TypeScript team has an explicit, long-standing position: **the compiler will not add lint rules.**

```ts
// tsc's job, stated precisely:
//   1. Parse TypeScript.
//   2. Check that types are internally CONSISTENT.
//   3. Erase types and emit JavaScript.
//
// "Consistent" is the operative word. tsc asks "does this assignment obey the
// rules of the type system?" — never "is this a good idea?"

// ── Things tsc reports ───────────────────────────────────────────────────────
const orderTotal: number = "1299";
// error TS2322: Type 'string' is not assignable to type 'number'.

interface Order { orderId: string; totalCents: number; }
const order: Order = { orderId: "ord_1" };
// error TS2741: Property 'totalCents' is missing.

function refund(order: Order) { return order.custmerId; }
// error TS2551: Property 'custmerId' does not exist. Did you mean 'customerId'?

// ── Things tsc does NOT report, and by design never will ─────────────────────

// 1. Floating promises. `void` is a perfectly legal thing to discard.
sendOrderConfirmationEmail(order);

// 2. `any` anywhere. `any` IS a type. Using it is legal by definition.
const payload: any = JSON.parse(rawBody);
payload.customer.address.postcode;   // tsc: fine. This can throw at runtime.

// 3. Conditions that are always true. Not a type error — just pointless.
const orderId: string = order.orderId;
if (orderId !== null) { /* always taken */ }

// 4. `async` with no `await`. Legal, and sometimes intentional.
async function computeTax(cents: number): Promise<number> { return cents * 0.2; }

// 5. Import style. `import { Order }` vs `import type { Order }` — identical
//    to the type system, wildly different at runtime under some bundlers.

// 6. Naming, ordering, complexity, dead code, forbidden APIs, `console.log`
//    in production, banning `require()` in ESM, enforcing your error class.
```

There are exactly **three** compiler flags that feel lint-like, and knowing them prevents duplicate configuration:

```jsonc
// tsconfig.json — these three ARE quasi-lint rules built into tsc
{
  "compilerOptions": {
    "noUnusedLocals": true,              // ~ @typescript-eslint/no-unused-vars
    "noUnusedParameters": true,          // ~ same rule, args option
    "noFallthroughCasesInSwitch": true,  // ~ no-fallthrough
    "noImplicitReturns": true            // ~ consistent-return
  }
}

// ⚠️ Recommendation: turn noUnusedLocals/noUnusedParameters OFF and let ESLint
//    own it. Reasons:
//      - tsc errors BLOCK the build; an unused variable mid-refactor shouldn't.
//      - tsc has no `argsIgnorePattern: "^_"` escape hatch. ESLint does.
//      - ESLint can auto-fix and can warn instead of error.
//    Pick one owner per concern. Two owners means two config files to argue with.
```

The division of labour, memorised:

```
tsc    →  "Are the types consistent?"        →  blocks compilation
ESLint →  "Given the types, is this wise?"   →  blocks the PR
```

Run **both** in CI. Neither subsumes the other.

### Concept 2 — What `typescript-eslint` actually is

Three packages, one meta-package:

```
typescript-eslint                         ← install THIS (v8+)
├── @typescript-eslint/parser             ← TS source  →  TSESTree AST
├── @typescript-eslint/eslint-plugin      ← ~130 rules
└── @typescript-eslint/typescript-estree  ← internals: AST + type-checker bridge
```

```ts
// ── What the parser does ─────────────────────────────────────────────────────
// TypeScript's own AST is NOT ESTree-shaped. ESLint rules are written against
// ESTree. The parser produces "TSESTree": ESTree plus TS-specific node types.
//
// Source:
interface OrderLineItem { sku: string; quantity: number; }
//
// Produces a node of type TSInterfaceDeclaration, containing TSPropertySignature
// nodes with TSStringKeyword / TSNumberKeyword type annotations.
//
// A stock ESLint rule that walks `Identifier` nodes still works — the shared
// ESTree parts are unchanged. That's why `no-unused-vars`-style core rules run
// at all on TS, and why `eqeqeq`, `no-console`, `curly` work with zero changes.

// ── The critical extra: the parser can also build a ts.Program ───────────────
// When you enable projectService/project, the parser doesn't just parse — it
// asks TypeScript to type-check the file, and attaches a `parserServices`
// object to the ESLint context. A rule can then do:
//
//   const services = ESLintUtils.getParserServices(context);
//   const type = services.getTypeAtLocation(node);
//   if (tsutils.isThenableType(checker, node, type)) { report(); }
//
// THAT is the mechanism behind every "TypeChecked" rule.
```

```ts
// ── Extension rules — the third category people miss ─────────────────────────
// Some core ESLint rules are simply WRONG on TypeScript. typescript-eslint
// ships replacements with the same name under its own namespace.
//
// You must disable the core one and enable the TS one. The shared configs
// already do this for you; if you hand-roll, don't forget.

// ❌ core rule flags the overload signature and the enum member as unused
"no-unused-vars": "error"

// ✅ TS-aware version understands overloads, enums, type params, declare
"no-unused-vars": "off",
"@typescript-eslint/no-unused-vars": "error"

// The full extension-rule family (all follow this off/on pattern):
//   no-unused-vars          no-shadow           no-redeclare
//   no-use-before-define    no-invalid-this     no-empty-function
//   dot-notation            require-await       no-implied-eval
//   default-param-last      no-array-constructor  no-loop-func
//   only-throw-error        prefer-promise-reject-errors  return-await
//   class-methods-use-this  max-params          no-magic-numbers
```

### Concept 3 — Flat config (`eslint.config.js`), and the legacy `.eslintrc`

ESLint v9 made **flat config** the default. It is a plain JS array of config objects, evaluated top to bottom, later objects overriding earlier ones for the files they match.

```js
// ── eslint.config.js — the mental model ──────────────────────────────────────
// It's an ARRAY. Each element is a config object. For a given file, ESLint:
//   1. walks the array in order,
//   2. keeps every object whose `files` glob matches (no `files` = matches all),
//   3. merges them left-to-right — later wins.
//
// No `extends`. No cascade through parent directories. No `env`. No plugin
// name resolution by string. Everything is an explicit JS value you imported.

import js from "@eslint/js";
import tseslint from "typescript-eslint";
import globals from "globals";

export default tseslint.config(
  // ① Global ignores — an object with ONLY `ignores` is special-cased as global.
  //    Paths are relative to the config file. `.eslintignore` is NOT read.
  { ignores: ["dist/**", "build/**", "coverage/**", "**/*.generated.ts"] },

  // ② Base config with no `files` → applies to every linted file.
  js.configs.recommended,

  // ③ Spread the TS preset (it's an array of several objects).
  ...tseslint.configs.recommendedTypeChecked,

  // ④ A targeted object.
  {
    files: ["src/**/*.ts"],
    languageOptions: {
      ecmaVersion: 2023,
      sourceType: "module",
      globals: { ...globals.node },      // replaces the old `env: { node: true }`
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname
      }
    },
    linterOptions: {
      reportUnusedDisableDirectives: "error"  // fail on stale eslint-disable comments
    },
    rules: {
      "@typescript-eslint/no-floating-promises": "error"
    }
  }
);
```

```jsonc
// ── The legacy .eslintrc equivalent, for reference ───────────────────────────
// You will meet this in existing repos. ESLint v8 and below default to it;
// v9 reads it ONLY with ESLINT_USE_FLAT_CONFIG=false, and v10 drops it entirely.
{
  "root": true,
  "parser": "@typescript-eslint/parser",         // ← a STRING, resolved by name
  "plugins": ["@typescript-eslint"],             // ← a STRING
  "extends": [                                   // ← cascade, not array merge
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended-type-checked",
    "plugin:@typescript-eslint/stylistic-type-checked"
  ],
  "parserOptions": {
    "project": ["./tsconfig.json"],              // no projectService in the old world
    "tsconfigRootDir": "__dirname"
  },
  "env": { "node": true, "es2022": true },       // → replaced by languageOptions.globals
  "ignorePatterns": ["dist", "coverage"],        // → replaced by the `ignores` object
  "overrides": [                                 // → replaced by extra array entries
    {
      "files": ["**/*.test.ts"],
      "rules": { "@typescript-eslint/no-unsafe-assignment": "off" }
    }
  ]
}
```

The translation table, which is all you need to migrate:

| `.eslintrc` | `eslint.config.js` |
|---|---|
| `extends: ["plugin:x/recommended"]` | `...xPlugin.configs.recommended` (imported) |
| `parser: "@typescript-eslint/parser"` | `languageOptions.parser: tseslint.parser` |
| `plugins: ["@typescript-eslint"]` | `plugins: { "@typescript-eslint": tseslint.plugin }` |
| `env: { node: true }` | `languageOptions.globals: globals.node` |
| `parserOptions` | `languageOptions.parserOptions` |
| `overrides: [{ files, rules }]` | another array entry with `files` and `rules` |
| `.eslintignore` | `{ ignores: [...] }` as the first array entry |
| `root: true` | implicit — flat config never cascades |

```bash
# Automated migration, then read the diff carefully:
npx @eslint/migrate-config .eslintrc.json
```

Two gotchas that bite immediately:

```js
// GOTCHA 1: eslint.config.js uses ESM `import` but your package.json says CommonJS.
// Fix either way:
//   a) "type": "module" in package.json, or
//   b) rename the file eslint.config.mjs
// (eslint.config.cjs with require() also works if you prefer.)

// GOTCHA 2: `ignores` inside an object that ALSO has `files` or `rules` is NOT
// global — it only excludes files from THAT object.
{ ignores: ["dist/**"] }                       // ✅ global ignore
{ files: ["**/*.ts"], ignores: ["dist/**"] }   // ❌ only scopes this one object
```

### Concept 4 — `recommended` vs `strict` vs `stylistic`, and the `*TypeChecked` variants

typescript-eslint ships its rules as **three orthogonal dimensions**, and the naming is systematic once you see the grid.

```
                  no type info              WITH type info
                  ───────────────────────   ────────────────────────────────
correctness,      recommended               recommendedTypeChecked
  low false-pos

correctness,      strict                    strictTypeChecked
  opinionated

formatting/       stylistic                 stylisticTypeChecked
  consistency
```

```ts
// ── The three tiers, by intent ───────────────────────────────────────────────

// recommended  — "things that are almost certainly bugs"
//   ~35 rules. Very low false-positive rate. The floor for any TS project.
//   e.g. no-explicit-any, no-misused-new, no-unsafe-function-type,
//        no-non-null-asserted-optional-chain, no-array-constructor

// strict       — recommended + "opinionated correctness"
//   ~+30 rules. Will flag code you consider fine. That's the point: it forces
//   you to defend it. Expect to disable 2–5 of them for your codebase.
//   e.g. no-non-null-assertion, no-unnecessary-condition,
//        no-dynamic-delete, prefer-reduce-type-parameter, no-invalid-void-type
//   ⚠️ NOT semver-stable in rule content: minor releases may add rules.
//      Pin your typescript-eslint version or expect new errors on `npm update`.

// stylistic    — "consistency, zero bug-catching"
//   ~+20 rules. Orthogonal to the other two — you compose it WITH one of them.
//   e.g. consistent-type-definitions, prefer-function-type, array-type,
//        consistent-indexed-object-style, non-nullable-type-assertion-style
//   ⚠️ Not a formatter. Does not replace Prettier. See "Going deeper".
```

```ts
// ── The *TypeChecked suffix ──────────────────────────────────────────────────
// Adding "TypeChecked" turns on rules that require a ts.Program. These are the
// rules that make the whole exercise worthwhile — and they cost real time.

// recommendedTypeChecked adds ~20 rules, including the ones you came for:
//   no-floating-promises          ← unhandled rejections
//   no-misused-promises           ← async handlers in sync positions
//   await-thenable                ← awaiting a non-promise
//   require-await                 ← async with no await
//   no-unsafe-assignment          ← assigning `any` to a typed slot
//   no-unsafe-member-access       ← reading a property off `any`
//   no-unsafe-call / -return / -argument
//   restrict-template-expressions ← `${someObject}` → "[object Object]"
//   unbound-method                ← passing a method and losing `this`

// strictTypeChecked adds on top:
//   no-unnecessary-condition      ← conditions that are always true/false
//   no-unnecessary-type-assertion ← `as X` where the type is already X
//   no-confusing-void-expression
//   prefer-nullish-coalescing / prefer-optional-chain
//   use-unknown-in-catch-callback-variable

// stylisticTypeChecked adds:
//   dot-notation (type-aware)
//   non-nullable-type-assertion-style
//   prefer-nullish-coalescing (stylistic framing)
```

```js
// ── Composing them — the four realistic setups ───────────────────────────────

// (A) Getting started / large legacy codebase you're migrating.
export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommended
);

// (B) The sensible default for a new backend service. ← START HERE
export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  { languageOptions: { parserOptions: { projectService: true,
      tsconfigRootDir: import.meta.dirname } } }
);

// (C) Maximum safety. Expect to spend an afternoon triaging, then love it.
export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  { languageOptions: { parserOptions: { projectService: true,
      tsconfigRootDir: import.meta.dirname } } }
);

// (D) Strict everywhere, but a few rules relaxed. This is where you actually
//     land after a week. Note the override object comes AFTER the presets.
export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: { parserOptions: { projectService: true,
      tsconfigRootDir: import.meta.dirname } },
    rules: {
      // Too noisy on a codebase that legitimately uses `!` after a guard fn:
      "@typescript-eslint/no-non-null-assertion": "off",
      // We use `interface` for public contracts and `type` for unions; the
      // stylistic rule would force one everywhere:
      "@typescript-eslint/consistent-type-definitions": "off"
    }
  }
);
```

### Concept 5 — Type-aware linting: `projectService` vs `parserOptions.project`

This is the single most important configuration decision, and the one that determines whether your lint run takes 3 seconds or 90.

```js
// ── Option A: projectService (typescript-eslint v8+) ← USE THIS ──────────────
{
  languageOptions: {
    parserOptions: {
      projectService: true,                  // discover tsconfig per file, like the IDE
      tsconfigRootDir: import.meta.dirname   // where to start looking
    }
  }
}

// How it works: it uses TypeScript's own **Project Service** — the same API the
// VS Code language server uses. For each file, it walks up the directory tree
// to find the nearest tsconfig.json, exactly as your editor does.
//
// ✅ No glob list to maintain. Add a new tsconfig, it's picked up.
// ✅ Monorepos work with zero extra config.
// ✅ Handles files NOT in any tsconfig (see allowDefaultProject below).
// ✅ Faster and lower-memory than `project` on large repos — it reuses one
//    service instead of building N independent Programs.
```

```js
// ── Option B: parserOptions.project (the older way) ──────────────────────────
{
  languageOptions: {
    parserOptions: {
      project: ["./tsconfig.json", "./tsconfig.test.json"],  // explicit list
      tsconfigRootDir: import.meta.dirname
    }
  }
}

// ❌ Every linted file MUST be `include`d by one of these tsconfigs, or:
//
//    error  Parsing error: ESLint was configured to run on `<tsconfigRootDir>/
//    vitest.config.ts` using `parserOptions.project`. However, that TSConfig
//    does not include this file.
//
// ❌ In a monorepo you end up listing ./packages/*/tsconfig.json and fighting
//    the globs. Each entry builds a separate Program → memory multiplies.
```

```js
// ── The escape hatch: files outside any tsconfig ─────────────────────────────
// Config files, scripts, .eslintrc-adjacent tooling — commonly not in `include`.
{
  languageOptions: {
    parserOptions: {
      projectService: {
        allowDefaultProject: ["*.js", "*.config.ts", "scripts/*.ts"],
        defaultProject: "tsconfig.json"
      },
      tsconfigRootDir: import.meta.dirname
    }
  }
}
// ⚠️ Keep allowDefaultProject SMALL (it's capped at 8 entries by default).
//    Each file in it gets its own inferred Program — expensive. The better fix
//    is usually to add those files to a tsconfig's `include`.
```

Now the cost, stated honestly:

```
Non-type-aware lint     : parse each file, run rules.        ~1–3 s for 500 files
Type-aware lint         : build a ts.Program (= a full tsc
                          type-check) THEN run rules.        ~20–90 s for 500 files
```

```ts
// The mental model: type-aware linting costs approximately
//
//     (time of `tsc --noEmit`)  +  (time of plain lint)
//
// because that is literally what happens. There is no way around it — the rules
// need the checker, and building the checker means checking the project.
//
// Mitigations that actually work:
//   1. Turn OFF type-aware rules for files that don't need them (tests? no —
//      tests need them MORE. Config files? yes.)
//   2. Cache: `eslint . --cache --cache-location .eslintcache`
//      ⚠️ Type-aware results depend on OTHER files, so the file-mtime cache can
//      go stale and miss cross-file violations. Safe locally, risky in CI.
//      Use --cache locally; never trust it as the only CI gate.
//   3. In CI, run lint and typecheck as PARALLEL jobs, not sequentially.
//   4. Don't lint dist/, coverage/, or generated code — ignore them globally.
//   5. Raise Node's heap on big monorepos:
//      NODE_OPTIONS=--max-old-space-size=8192 npx eslint .
```

```js
// ── Disabling type-aware rules for the files that can't support them ─────────
// tseslint ships a preset for exactly this. Put it LAST so it wins.
export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  {
    files: ["**/*.js", "**/*.mjs", "*.config.ts"],
    extends: [tseslint.configs.disableTypeChecked]   // turns off every type-aware rule
  }
);
```

### Concept 6 — The backend-critical rules, one by one

These seven earn their keep in any Node service. Learn what each actually detects.

```ts
// ─────────────────────────────────────────────────────────────────────────────
// 1. @typescript-eslint/no-floating-promises          [type-aware]
//    "A Promise was created and nothing handles its rejection."
//    Severity in Node ≥15: unhandled rejection = process exit. This is a
//    crash-the-server rule, not a style rule.
// ─────────────────────────────────────────────────────────────────────────────
async function chargeCard(orderId: string): Promise<void> { /* ... */ }

// ❌ flagged
chargeCard(orderId);

// ✅ four legitimate resolutions, in order of preference:
await chargeCard(orderId);                        // 1. actually wait for it
chargeCard(orderId).catch(handleChargeFailure);   // 2. handle the rejection
void chargeCard(orderId);                         // 3. explicit fire-and-forget
                                                  //    (allowed by default; the
                                                  //     `void` documents intent)
try { await chargeCard(orderId); } catch (err) { logger.error({ err }); }  // 4.

// Options worth knowing:
"@typescript-eslint/no-floating-promises": ["error", {
  ignoreVoid: true,        // default true — `void promise` is accepted
  ignoreIIFE: false,       // default false — flag `(async () => {})()` too
  checkThenables: true     // also flag non-Promise thenables (knex query builders!)
}]

// ─────────────────────────────────────────────────────────────────────────────
// 2. @typescript-eslint/no-misused-promises          [type-aware]
//    "A Promise was passed somewhere a non-Promise was expected."
//    Catches the #1 Express bug and the #1 array-callback bug.
// ─────────────────────────────────────────────────────────────────────────────

// ❌ Express's RequestHandler returns void. Your async fn returns Promise<void>.
//    Express never awaits it → rejection escapes → request hangs, or crashes.
app.get("/orders/:orderId", async (req, res) => {
  const order = await orderRepository.findById(req.params.orderId);
  res.json(order);
});

// ✅ wrap it — the wrapper forwards rejections to Express's error middleware
const asyncHandler =
  <T extends RequestHandler>(fn: T): RequestHandler =>
  (req, res, next) => { void Promise.resolve(fn(req, res, next)).catch(next); };

app.get("/orders/:orderId", asyncHandler(async (req, res) => {
  const order = await orderRepository.findById(req.params.orderId);
  res.json(order);
}));
// (Express 5 handles async handlers natively — but the rule still fires on
//  Express 4 types, and on every other sync-callback API.)

// ❌ a Promise is always truthy — this branch ALWAYS runs
if (isOrderRefundable(orderId)) { /* isOrderRefundable is async */ }

// ❌ .filter expects boolean; every element survives
const active = orders.filter(async (o) => await isActive(o.orderId));

// ❌ forEach doesn't await — the loop finishes before any write completes
orders.forEach(async (o) => { await auditLog.write(o); });
// ✅
for (const o of orders) { await auditLog.write(o); }
// ✅ or, for parallelism:
await Promise.all(orders.map((o) => auditLog.write(o)));

// ─────────────────────────────────────────────────────────────────────────────
// 3. @typescript-eslint/consistent-type-imports      [syntax-only, auto-fixable]
//    "Imports used only as types must say so."
// ─────────────────────────────────────────────────────────────────────────────

// ❌ a runtime import of a module you only needed types from
import { OrderRepository, type Order } from "./orderRepository.js";
import { PaymentProvider } from "./paymentProvider.js";   // only used as a type

// ✅
import { OrderRepository } from "./orderRepository.js";
import type { Order } from "./orderRepository.js";
import type { PaymentProvider } from "./paymentProvider.js";

// Why it matters on a backend, concretely:
//   - `verbatimModuleSyntax: true` in tsconfig makes tsc emit imports EXACTLY as
//     written. A plain `import { PaymentProvider }` becomes a real require()
//     of a module you never needed → slower cold start, and possible
//     circular-import crash at runtime even though types were acyclic.
//   - Decorator metadata (NestJS/TypeORM) behaves differently for type-only
//     imports — this rule keeps that boundary explicit.
//   - It is auto-fixable. `eslint --fix` cleans an entire codebase in seconds.

"@typescript-eslint/consistent-type-imports": ["error", {
  prefer: "type-imports",              // vs "no-type-imports"
  fixStyle: "inline-type-imports",     // `import { type Order }` — keeps one
                                       //  statement instead of splitting
  disallowTypeAnnotations: true
}]
// Pair with the export side:
"@typescript-eslint/consistent-type-exports": "error"

// ─────────────────────────────────────────────────────────────────────────────
// 4. @typescript-eslint/no-explicit-any              [syntax-only]
//    "You wrote `any`. Every guarantee downstream of this point is void."
// ─────────────────────────────────────────────────────────────────────────────

// ❌
function parseWebhook(rawBody: any) {
  return rawBody.event.type;       // no check. Throws at 3am. tsc says nothing.
}

// ✅ `unknown` forces you to narrow before use
function parseWebhook(rawBody: unknown): string {
  const parsed = webhookSchema.parse(rawBody);   // zod / valibot / io-ts
  return parsed.event.type;
}

"@typescript-eslint/no-explicit-any": ["error", {
  fixToUnknown: false,       // true = --fix rewrites any → unknown (will break
                             //         compilation; useful as a migration step)
  ignoreRestArgs: true       // allow `...args: any[]` in logger/wrapper signatures
}]

// Note: this rule only catches `any` you WRITE. `any` that arrives from an
// untyped library is caught by the no-unsafe-* family instead:
//   no-unsafe-assignment / -member-access / -call / -return / -argument
// Those are the ones that actually stop `any` propagating. Keep them on.

// ─────────────────────────────────────────────────────────────────────────────
// 5. @typescript-eslint/no-unnecessary-condition     [type-aware, strict tier]
//    "This condition's outcome is determined by the types."
//    The best dead-code detector you will ever run.
// ─────────────────────────────────────────────────────────────────────────────

interface Order { orderId: string; couponCode: string | null; }

function describe(order: Order): string {
  // ❌ orderId is `string` — never null. This guard is dead code, and it lies
  //    to the next reader about what can be null.
  if (order.orderId != null) { /* ... */ }

  // ❌ optional chaining on a non-nullable value
  const len = order.orderId?.length;

  // ✅ couponCode genuinely is `string | null`
  if (order.couponCode !== null) { return order.couponCode; }
  return "none";
}

// This rule is how you find the leftovers after tightening a type. When you
// change `couponCode: string | null` → `couponCode: string`, tsc stays silent
// about the now-pointless guards. This rule lists every one of them.
//
// ⚠️ REQUIRES strictNullChecks. Without it, every type includes null and the
//    rule refuses to run (it will tell you so).
// ⚠️ Produces false positives at genuine trust boundaries — data from
//    JSON.parse, a DB driver, or process.env may not match its declared type.
//    Fix the type (or validate), don't disable the rule.

"@typescript-eslint/no-unnecessary-condition": ["error", {
  allowConstantLoopConditions: true   // permit `while (true)`
}]

// ─────────────────────────────────────────────────────────────────────────────
// 6. @typescript-eslint/strict-boolean-expressions   [type-aware, NOT in any preset]
//    "Only booleans in boolean positions."
//    Opinionated. Divisive. Genuinely prevents a whole bug class.
// ─────────────────────────────────────────────────────────────────────────────

function summarise(order: {
  couponCode: string | null;
  discountCents: number;
  notes?: string;
}): void {
  // ❌ "" is falsy. A coupon code of "" takes the else branch. Probably wrong.
  if (order.couponCode) { /* ... */ }
  // ✅ explicit
  if (order.couponCode !== null && order.couponCode !== "") { /* ... */ }

  // ❌ 0 is falsy. A zero discount is a REAL, valid discount.
  //    This is the classic pagination/quantity/price bug.
  if (order.discountCents) { /* ... */ }
  // ✅
  if (order.discountCents > 0) { /* ... */ }

  // ❌ conflates "absent" with "empty string"
  if (order.notes) { /* ... */ }
  // ✅
  if (order.notes !== undefined && order.notes.length > 0) { /* ... */ }
}

"@typescript-eslint/strict-boolean-expressions": ["error", {
  allowString: false,           // ← the important one: bans truthy-string checks
  allowNumber: false,           // ← bans truthy-number checks
  allowNullableObject: true,    // `if (user)` on `User | null` stays legal
  allowNullableBoolean: false,
  allowNullableString: false,
  allowNullableNumber: false,
  allowAny: false
}]
// Pragmatic middle ground for an existing codebase: allowNullableObject: true,
// allowString: false, allowNumber: false. That kills the "" and 0 bugs — which
// are the ones that actually cost money — without banning `if (user)`.

// ─────────────────────────────────────────────────────────────────────────────
// 7. @typescript-eslint/require-await                [type-aware, extension rule]
//    "async with no await is a lie about the function's nature."
// ─────────────────────────────────────────────────────────────────────────────

// ❌ pointless Promise wrapper; also forces every caller to await
async function formatOrderNumber(orderId: string): Promise<string> {
  return `ORD-${orderId.toUpperCase()}`;
}
// ✅
function formatOrderNumber(orderId: string): string {
  return `ORD-${orderId.toUpperCase()}`;
}

// ✅ ALSO fine — deliberately async to satisfy an interface. `async` here is
//    intentional, so return an explicit Promise instead and drop `async`:
function healthCheck(): Promise<HealthStatus> {
  return Promise.resolve({ status: "ok" });
}

// ⚠️ Remember to turn OFF the core rule — it can't see return types:
"require-await": "off",
"@typescript-eslint/require-await": "error"

// Companion rule, often more valuable in a backend:
"@typescript-eslint/return-await": ["error", "in-try-catch"]
// `return await` inside try/catch is REQUIRED (otherwise the catch never fires);
// outside try/catch it's a redundant microtask. This rule enforces exactly that.
```

### Concept 7 — Per-file overrides

Flat config has no `overrides` key. You just add another object with a `files` glob. Later objects win.

```js
export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,

  // ── Tests: relax the `any` police, keep the promise rules ──────────────────
  {
    files: ["**/*.test.ts", "**/*.spec.ts", "test/**/*.ts"],
    rules: {
      // Mocks and fixtures legitimately produce `any`. Fighting this in tests
      // costs more than it saves.
      "@typescript-eslint/no-explicit-any": "off",
      "@typescript-eslint/no-unsafe-assignment": "off",
      "@typescript-eslint/no-unsafe-member-access": "off",
      "@typescript-eslint/no-unsafe-call": "off",
      "@typescript-eslint/no-unsafe-argument": "off",
      // `expect(order.customer!.email)` is fine in a test where you've asserted
      // the shape two lines above.
      "@typescript-eslint/no-non-null-assertion": "off",
      // Test data is full of literal numbers.
      "@typescript-eslint/no-magic-numbers": "off",
      // ⚠️ DELIBERATELY NOT DISABLED — a floating promise in a test is the
      //    single most common cause of a flaky test that passes locally and
      //    fails in CI. Keep these ON in tests, always:
      //      no-floating-promises, no-misused-promises, await-thenable
    }
  },

  // ── Config & build files: no type-aware rules (not in any tsconfig) ────────
  {
    files: ["*.config.ts", "*.config.js", "scripts/**/*.ts"],
    extends: [tseslint.configs.disableTypeChecked],
    rules: {
      "no-console": "off"   // scripts print. That's their job.
    }
  },

  // ── Generated code: don't lint it at all ──────────────────────────────────
  { ignores: ["src/generated/**", "prisma/client/**", "**/*.pb.ts"] },

  // ── Migrations: sequential awaits in loops are correct here ───────────────
  {
    files: ["src/db/migrations/**/*.ts"],
    rules: { "no-await-in-loop": "off" }
  },

  // ── Plain JS that survives in the repo ────────────────────────────────────
  {
    files: ["**/*.js", "**/*.cjs", "**/*.mjs"],
    extends: [tseslint.configs.disableTypeChecked],
    rules: { "@typescript-eslint/no-require-imports": "off" }
  }
);
```

```js
// ── Glob semantics you must get right ─────────────────────────────────────────
"**/*.test.ts"      // any depth                          ✅ what you almost always want
"*.test.ts"         // ROOT LEVEL ONLY                    ❌ misses src/**/x.test.ts
"src/**/*.ts"       // everything under src, any depth
"test/*.ts"         // direct children of test/ only
["**/*.test.ts", "**/*.spec.ts", "**/__tests__/**/*.ts"]   // an array is OR
```

### Concept 8 — Disabling rules properly

There is a hierarchy of correctness here, from best to worst.

```ts
// ── LEVEL 1 (best): fix the code ─────────────────────────────────────────────
// ❌
const order = JSON.parse(rawBody);          // any → no-unsafe-assignment
// ✅
const order = orderSchema.parse(JSON.parse(rawBody));   // typed, validated

// ── LEVEL 2: disable ONE rule on ONE line, WITH A REASON ─────────────────────
// The `--` separator makes everything after it a description. ESLint supports
// this natively, and `reportUnusedDisableDirectives` will tell you when the
// comment becomes obsolete.

// eslint-disable-next-line @typescript-eslint/no-explicit-any -- legacy-payments-sdk@2 ships no types; see JIRA-4821
const paymentsClient: any = new LegacyPaymentsSDK(apiKey);

// ── LEVEL 3: disable a rule for a whole FILE, via config not comments ────────
// Better than a file-top comment because it's discoverable, reviewable, and
// shows up in a config diff instead of being buried on line 1 of a source file.
{
  files: ["src/integrations/legacyPayments.ts"],
  rules: { "@typescript-eslint/no-unsafe-member-access": "off" }
}

// ── LEVEL 4 (avoid): a blanket file-level comment ────────────────────────────
/* eslint-disable */                        // ❌ every rule, whole file, forever
/* eslint-disable @typescript-eslint/no-unsafe-member-access */   // ⚠️ at least scoped

// ── LEVEL 5 (never): turning the rule off globally because it's noisy ────────
// If a rule fires 200 times, that's 200 findings, not a bad rule. Set it to
// "warn", fix incrementally, then promote to "error".
```

```js
// ── Make stale disables an error ─────────────────────────────────────────────
// The #1 rot problem: someone fixes the code but leaves the disable comment,
// and it silently suppresses a REAL violation introduced two years later.
{
  linterOptions: {
    reportUnusedDisableDirectives: "error"
  }
}
// CLI equivalent: eslint . --report-unused-disable-directives

// ── Require a reason on every disable ────────────────────────────────────────
// npm i -D eslint-plugin-eslint-comments  (or use the built-in rule below)
{
  rules: {
    "@eslint-community/eslint-comments/require-description": ["error", {
      ignore: ["eslint-enable"]
    }],
    "@eslint-community/eslint-comments/no-unlimited-disable": "error",  // bans bare /* eslint-disable */
    "@eslint-community/eslint-comments/no-unused-disable": "error"
  }
}
```

```ts
// ── @ts-expect-error vs @ts-ignore vs eslint-disable ─────────────────────────
// Different tools, frequently confused. Two of these talk to tsc, one to ESLint.

// @ts-ignore        — suppress a TS error. If the error goes away, silently no-ops.
// @ts-expect-error  — suppress a TS error AND fail if there is no error to suppress.
//                     Strictly better. Enforce it:
"@typescript-eslint/ban-ts-comment": ["error", {
  "ts-expect-error": "allow-with-description",   // must explain yourself
  "ts-ignore": true,                             // banned outright
  "ts-nocheck": true,
  "ts-check": false,
  minimumDescriptionLength: 10
}]

// @ts-expect-error -- upstream @types/express@4 types req.user as any; fixed in v5
const userId: string = req.user.id;
```

---

## Example 1 — basic

A minimal, complete, working setup for a small TypeScript Node project.

```bash
# ── Scaffold ─────────────────────────────────────────────────────────────────
mkdir ts-lint-lab && cd ts-lint-lab
npm init -y
npm pkg set type=module
npm i -D eslint typescript typescript-eslint @types/node
```

```jsonc
// ── tsconfig.json ───────────────────────────────────────────────────────────
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,               // REQUIRED for no-unnecessary-condition and
                                  // strict-boolean-expressions to work at all
    "verbatimModuleSyntax": true, // makes consistent-type-imports load-bearing
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmitOnError": true
  },
  "include": ["src/**/*.ts", "eslint.config.js"],   // ← include the config file so
                                                    //   projectService can type it
  "exclude": ["node_modules", "dist"]
}
```

```js
// ── eslint.config.js ────────────────────────────────────────────────────────
// @ts-check   ← gives you type errors in this file itself. Worth it.
import js from "@eslint/js";
import globals from "globals";
import tseslint from "typescript-eslint";

export default tseslint.config(
  // ① Global ignores. Must be its own object with ONLY `ignores`.
  {
    ignores: ["dist/**", "coverage/**", "node_modules/**"]
  },

  // ② ESLint's own baseline JS rules.
  js.configs.recommended,

  // ③ TypeScript rules WITH type information.
  ...tseslint.configs.recommendedTypeChecked,

  // ④ Stylistic consistency (composes with the above).
  ...tseslint.configs.stylisticTypeChecked,

  // ⑤ Our layer: language options + rule tuning.
  {
    files: ["**/*.ts"],
    languageOptions: {
      ecmaVersion: 2022,
      sourceType: "module",
      globals: { ...globals.node },          // process, Buffer, __dirname, ...
      parserOptions: {
        projectService: true,                // ← enables type-aware rules
        tsconfigRootDir: import.meta.dirname // ← where to find tsconfig.json
      }
    },
    linterOptions: {
      reportUnusedDisableDirectives: "error" // stale eslint-disable = failure
    },
    rules: {
      // The backend-critical set, made explicit so it's obvious they're on:
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/consistent-type-imports": ["error", {
        prefer: "type-imports",
        fixStyle: "inline-type-imports"
      }],
      "@typescript-eslint/no-unused-vars": ["error", {
        argsIgnorePattern: "^_",             // `(_req, res) =>` is fine
        varsIgnorePattern: "^_",
        caughtErrorsIgnorePattern: "^_"
      }],
      // Core rules that still matter:
      eqeqeq: ["error", "always", { null: "ignore" }],  // `x != null` stays legal
      "no-console": ["warn", { allow: ["warn", "error"] }]
    }
  },

  // ⑥ The config file itself is JS, not in a TS program.
  {
    files: ["eslint.config.js"],
    extends: [tseslint.configs.disableTypeChecked]
  }
);
```

```ts
// ── src/orderService.ts — deliberately contains one violation of each rule ──
import { setTimeout as sleep } from "node:timers/promises";
import { OrderRepository } from "./orderRepository.js";
import { Order } from "./orderRepository.js";   // ← used only as a type

export class OrderService {
  constructor(private readonly orderRepository: OrderRepository) {}

  async cancelOrder(orderId: string, reason: any): Promise<void> {
    const order = await this.orderRepository.findById(orderId);
    if (order === null) throw new Error(`order ${orderId} not found`);

    this.notifyCustomer(order, reason);     // ← floating promise
    await this.orderRepository.markCancelled(orderId);
  }

  private async notifyCustomer(order: Order, reason: string): Promise<void> {
    await sleep(10);
    console.log(`cancelled ${order.orderId}: ${reason}`);
  }
}
```

```
$ npx eslint .

/ts-lint-lab/src/orderService.ts
   4:10  error  All imports in the declaration are only used as types.
                Use `import type`
                @typescript-eslint/consistent-type-imports
   9:44  error  Unexpected any. Specify a different type
                @typescript-eslint/no-explicit-any
  13:5   error  Promises must be awaited, end with a call to .catch, end with a
                call to .then with a rejection handler or be explicitly marked
                as ignored with the `void` operator
                @typescript-eslint/no-floating-promises
  20:5   warn   Unexpected console statement
                no-console

✖ 4 problems (3 errors, 1 warning)
  1 error potentially fixable with the `--fix` option.
```

```bash
$ npx eslint . --fix
# Fixes the import automatically:
#   -import { Order } from "./orderRepository.js";
#   +import type { Order } from "./orderRepository.js";
# The other three require human judgement and are left alone.
```

```ts
// ── src/orderService.ts — corrected ─────────────────────────────────────────
import { setTimeout as sleep } from "node:timers/promises";
import { OrderRepository } from "./orderRepository.js";
import type { Order } from "./orderRepository.js";   // ✅ type-only

export class OrderService {
  constructor(private readonly orderRepository: OrderRepository) {}

  async cancelOrder(orderId: string, reason: string): Promise<void> {   // ✅ no any
    const order = await this.orderRepository.findById(orderId);
    if (order === null) throw new Error(`order ${orderId} not found`);

    await this.notifyCustomer(order, reason);   // ✅ awaited
    await this.orderRepository.markCancelled(orderId);
  }

  private async notifyCustomer(order: Order, reason: string): Promise<void> {
    await sleep(10);
    logger.info({ orderId: order.orderId, reason }, "order cancelled");  // ✅
  }
}
```

---

## Example 2 — real world backend use case

A complete lint setup for a production Express + Postgres service in a small monorepo, with tests, scripts, migrations, CI, and pre-commit hooks.

```
api/
├── eslint.config.js
├── tsconfig.json                 # src/
├── tsconfig.test.json            # test/ + src/
├── package.json
├── src/
│   ├── server.ts
│   ├── config.ts
│   ├── routes/orderRoutes.ts
│   ├── services/orderService.ts
│   ├── repositories/orderRepository.ts
│   ├── db/migrations/001_create_orders.ts
│   └── generated/prismaTypes.ts  # generated — never linted
├── test/
│   └── orderService.test.ts
├── scripts/
│   └── backfillOrderTotals.ts
└── .github/workflows/ci.yml
```

```js
// ── eslint.config.js — the production configuration ──────────────────────────
// @ts-check
import js from "@eslint/js";
import globals from "globals";
import tseslint from "typescript-eslint";
import importPlugin from "eslint-plugin-import-x";
import unicorn from "eslint-plugin-unicorn";
import vitest from "@vitest/eslint-plugin";

export default tseslint.config(
  // ═══════════════════════════════════════════════════════════════════════════
  // ① GLOBAL IGNORES — nothing below this ever sees these paths.
  //    Generated code and build output are the two big lint-time wins: they're
  //    often the largest files in the repo and linting them is pure waste.
  // ═══════════════════════════════════════════════════════════════════════════
  {
    ignores: [
      "dist/**",
      "coverage/**",
      "node_modules/**",
      "src/generated/**",         // prisma / graphql-codegen / protobuf output
      "**/*.d.ts",                // hand-written .d.ts are fine; generated ones aren't
      ".eslintcache"
    ]
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // ② BASELINE — plain-JS correctness rules.
  // ═══════════════════════════════════════════════════════════════════════════
  js.configs.recommended,

  // ═══════════════════════════════════════════════════════════════════════════
  // ③ TYPESCRIPT — strict tier, type-aware, plus stylistic.
  //    strictTypeChecked is the right default for a NEW service. For a legacy
  //    one, start at recommendedTypeChecked and promote later.
  // ═══════════════════════════════════════════════════════════════════════════
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,

  // ═══════════════════════════════════════════════════════════════════════════
  // ④ LANGUAGE OPTIONS — where type-aware linting is switched on.
  // ═══════════════════════════════════════════════════════════════════════════
  {
    files: ["**/*.ts"],
    languageOptions: {
      ecmaVersion: 2023,
      sourceType: "module",
      globals: { ...globals.node },
      parserOptions: {
        // projectService walks up from each file to the nearest tsconfig.json,
        // exactly like the VS Code language server. Handles src/ (tsconfig.json)
        // and test/ (tsconfig.test.json) with no glob list to maintain.
        projectService: {
          // Files not covered by ANY tsconfig get an inferred default project.
          // Keep this list short — each entry is expensive.
          allowDefaultProject: ["*.js", "*.config.ts"],
          defaultProject: "tsconfig.json"
        },
        tsconfigRootDir: import.meta.dirname
      }
    },
    linterOptions: {
      reportUnusedDisableDirectives: "error"
    },
    plugins: {
      "import-x": importPlugin,
      unicorn
    },
    rules: {
      // ───────────────────────────────────────────────────────────────────────
      // ASYNC SAFETY — the rules that keep the process alive.
      // ───────────────────────────────────────────────────────────────────────
      "@typescript-eslint/no-floating-promises": ["error", {
        ignoreVoid: true,        // `void fireAndForget()` is an accepted opt-out
        ignoreIIFE: false,
        checkThenables: true     // knex/kysely builders are thenable, not Promise
      }],
      "@typescript-eslint/no-misused-promises": ["error", {
        checksVoidReturn: {
          arguments: true,       // catches async Express handlers
          attributes: true,
          properties: true,
          returns: true,
          variables: true
        },
        checksConditionals: true, // catches `if (asyncFn())`
        checksSpreads: true
      }],
      "@typescript-eslint/await-thenable": "error",
      "@typescript-eslint/require-await": "error",
      "@typescript-eslint/return-await": ["error", "in-try-catch"],
      "@typescript-eslint/promise-function-async": "error",
      "@typescript-eslint/no-await-in-loop": "off",   // sequential DB writes are
                                                      // often deliberate here

      // ───────────────────────────────────────────────────────────────────────
      // TYPE SAFETY — stop `any` from leaking in from untyped deps.
      // ───────────────────────────────────────────────────────────────────────
      "@typescript-eslint/no-explicit-any": ["error", { ignoreRestArgs: true }],
      "@typescript-eslint/no-unsafe-assignment": "error",
      "@typescript-eslint/no-unsafe-member-access": "error",
      "@typescript-eslint/no-unsafe-call": "error",
      "@typescript-eslint/no-unsafe-return": "error",
      "@typescript-eslint/no-unsafe-argument": "error",
      "@typescript-eslint/no-non-null-assertion": "error",
      "@typescript-eslint/no-unnecessary-condition": ["error", {
        allowConstantLoopConditions: true
      }],
      "@typescript-eslint/no-unnecessary-type-assertion": "error",
      "@typescript-eslint/restrict-template-expressions": ["error", {
        allowNumber: true,       // `${count}` is fine
        allowBoolean: false,
        allowNullish: false,     // `${maybeNull}` printing "null" is a bug
        allowAny: false
      }],

      // ───────────────────────────────────────────────────────────────────────
      // BOOLEAN DISCIPLINE — the "" and 0 bug class.
      // Not in any preset; opt in deliberately.
      // ───────────────────────────────────────────────────────────────────────
      "@typescript-eslint/strict-boolean-expressions": ["error", {
        allowString: false,        // no truthy-string checks
        allowNumber: false,        // no truthy-number checks
        allowNullableObject: true, // `if (order)` on `Order | null` stays legal
        allowNullableBoolean: false,
        allowNullableString: false,
        allowNullableNumber: false,
        allowAny: false
      }],

      // ───────────────────────────────────────────────────────────────────────
      // IMPORTS — bundle size, cold start, and circular-import safety.
      // ───────────────────────────────────────────────────────────────────────
      "@typescript-eslint/consistent-type-imports": ["error", {
        prefer: "type-imports",
        fixStyle: "inline-type-imports",
        disallowTypeAnnotations: true
      }],
      "@typescript-eslint/consistent-type-exports": ["error", {
        fixMixedExportsWithInlineTypeSpecifier: true
      }],
      "@typescript-eslint/no-import-type-side-effects": "error",
      "import-x/no-cycle": ["error", { maxDepth: 6 }],   // circular imports crash at runtime
      "import-x/no-extraneous-dependencies": ["error", {
        devDependencies: ["**/*.test.ts", "scripts/**", "*.config.ts"]
      }],
      "import-x/order": ["error", {
        groups: ["builtin", "external", "internal", "parent", "sibling", "index"],
        "newlines-between": "always",
        alphabetize: { order: "asc", caseInsensitive: true }
      }],

      // ───────────────────────────────────────────────────────────────────────
      // ERROR HANDLING — see doc 60.
      // ───────────────────────────────────────────────────────────────────────
      "@typescript-eslint/only-throw-error": "error",        // `throw "oops"` banned
      "@typescript-eslint/prefer-promise-reject-errors": "error",
      "@typescript-eslint/use-unknown-in-catch-callback-variable": "error",
      "@typescript-eslint/no-unnecessary-type-parameters": "error",

      // ───────────────────────────────────────────────────────────────────────
      // HOUSE STYLE
      // ───────────────────────────────────────────────────────────────────────
      "@typescript-eslint/naming-convention": ["error",
        { selector: "typeLike", format: ["PascalCase"] },
        { selector: "interface", format: ["PascalCase"], custom: {
            regex: "^I[A-Z]", match: false } },     // ban IOrderService
        { selector: "variable", format: ["camelCase", "UPPER_CASE"],
          leadingUnderscore: "allow" },
        { selector: "function", format: ["camelCase"] },
        { selector: "enumMember", format: ["UPPER_CASE"] }
      ],
      "@typescript-eslint/explicit-module-boundary-types": "off",  // inference is good
      "@typescript-eslint/no-unused-vars": ["error", {
        args: "after-used",
        argsIgnorePattern: "^_",
        varsIgnorePattern: "^_",
        caughtErrorsIgnorePattern: "^_",
        destructuredArrayIgnorePattern: "^_"
      }],
      "no-console": ["error", { allow: [] }],   // use the structured logger
      "no-restricted-imports": ["error", {
        paths: [
          { name: "lodash", message: "Import the submodule: lodash/pick" }
        ],
        patterns: [
          { group: ["../../*"], message: "Use the '@/' path alias instead" }
        ]
      }],
      "no-restricted-syntax": ["error", {
        selector: "TSEnumDeclaration",
        message: "Use a union of string literals instead of enum — see doc 69."
      }],
      "unicorn/prefer-node-protocol": "error"   // `node:fs` not `fs`
    }
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // ⑤ TESTS — relax `any` hygiene, KEEP the async rules.
  // ═══════════════════════════════════════════════════════════════════════════
  {
    files: ["test/**/*.ts", "**/*.test.ts", "**/*.spec.ts"],
    plugins: { vitest },
    rules: {
      ...vitest.configs.recommended.rules,

      // Mock factories and fixtures produce `any` by nature. Fighting it here
      // adds noise without adding safety.
      "@typescript-eslint/no-explicit-any": "off",
      "@typescript-eslint/no-unsafe-assignment": "off",
      "@typescript-eslint/no-unsafe-member-access": "off",
      "@typescript-eslint/no-unsafe-call": "off",
      "@typescript-eslint/no-unsafe-argument": "off",
      // `expect(order!.totalCents)` after an assertion is idiomatic.
      "@typescript-eslint/no-non-null-assertion": "off",
      // Test data is literal numbers all the way down.
      "@typescript-eslint/no-magic-numbers": "off",
      // Partial mocks fail no-unnecessary-condition constantly.
      "@typescript-eslint/no-unnecessary-condition": "off",
      // Deliberate `as unknown as X` casts for stubs.
      "@typescript-eslint/no-unsafe-type-assertion": "off",

      // ⚠️ NOT DISABLED, ON PURPOSE. A forgotten `await expect(...)` is the
      //    number-one source of tests that pass locally and flake in CI.
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      "@typescript-eslint/await-thenable": "error",
      "vitest/expect-expect": "error",
      "vitest/no-focused-tests": "error",       // `.only` must never reach main
      "vitest/no-disabled-tests": "warn",
      "vitest/valid-expect": "error"
    }
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // ⑥ MIGRATIONS — different rules apply to schema code.
  // ═══════════════════════════════════════════════════════════════════════════
  {
    files: ["src/db/migrations/**/*.ts"],
    rules: {
      // Migrations MUST run statements in order. Parallelising is a bug.
      "no-await-in-loop": "off",
      // Raw SQL strings are long and contain interpolations by design.
      "@typescript-eslint/restrict-template-expressions": "off",
      // Migration filenames are numeric-prefixed; naming rules don't apply.
      "@typescript-eslint/naming-convention": "off"
    }
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // ⑦ SCRIPTS / CONFIG — outside the main tsconfig, allow printing.
  // ═══════════════════════════════════════════════════════════════════════════
  {
    files: ["scripts/**/*.ts", "*.config.ts", "*.config.js"],
    rules: {
      "no-console": "off",
      "import-x/no-extraneous-dependencies": "off",
      "@typescript-eslint/no-restricted-imports": "off"
    }
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // ⑧ PLAIN JS — cannot support type-aware rules. Must be LAST to win.
  // ═══════════════════════════════════════════════════════════════════════════
  {
    files: ["**/*.js", "**/*.cjs", "**/*.mjs"],
    extends: [tseslint.configs.disableTypeChecked],
    rules: {
      "@typescript-eslint/no-require-imports": "off",
      "@typescript-eslint/no-var-requires": "off"
    }
  }
);
```

```jsonc
// ── tsconfig.test.json — so test/ has a project for type-aware linting ──────
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": true,
    "types": ["node", "vitest/globals"]
  },
  // MUST include both, or the tests can't see the source types.
  "include": ["src/**/*.ts", "test/**/*.ts", "**/*.test.ts"]
}
```

```jsonc
// ── package.json ────────────────────────────────────────────────────────────
{
  "type": "module",
  "scripts": {
    "typecheck": "tsc --noEmit -p tsconfig.json",
    "typecheck:test": "tsc --noEmit -p tsconfig.test.json",

    // Local: cached, fast on the second run.
    "lint": "eslint . --cache --cache-location node_modules/.cache/.eslintcache",
    "lint:fix": "npm run lint -- --fix",

    // CI: no cache (type-aware results are cross-file; a stale cache can hide
    // a real violation), and warnings are failures.
    "lint:ci": "eslint . --max-warnings 0 --format=stylish",

    "format": "prettier --write .",
    "format:check": "prettier --check .",

    // The one command a developer runs before pushing.
    "check": "npm run typecheck && npm run lint:ci && npm run format:check"
  }
}
```

```yaml
# ── .github/workflows/ci.yml — lint as a first-class gate ───────────────────
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  # Run typecheck and lint as SEPARATE, PARALLEL jobs. They're independent, both
  # take ~30-60s on a real service, and a developer wants to see BOTH failures
  # in one CI run — not fix the type error, push, and then discover 40 lint errors.
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: "npm" }
      - run: npm ci
      - run: npm run typecheck
      - run: npm run typecheck:test

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: "npm" }
      - run: npm ci

      # Type-aware linting builds a full ts.Program. On a large monorepo the
      # default 4GB Node heap is not enough and you get an opaque OOM.
      - name: Lint
        run: npx eslint . --max-warnings 0
        env:
          NODE_OPTIONS: "--max-old-space-size=6144"

      # Inline PR annotations on the changed lines. Optional but transforms
      # the review experience — errors appear in the diff, not in a log.
      - name: Annotate PR
        if: always()
        run: npx eslint . --format=@microsoft/eslint-formatter-sarif -o eslint.sarif
        continue-on-error: true
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: { sarif_file: eslint.sarif }

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: "npm" }
      - run: npm ci
      - run: npm test
```

```jsonc
// ── .lintstagedrc.json + husky — catch it before it reaches CI ──────────────
// npm i -D husky lint-staged
// npx husky init
// echo "npx lint-staged" > .husky/pre-commit
{
  "*.{ts,tsx}": [
    // --no-warn-ignored: lint-staged passes explicit paths, and ESLint errors
    // if you name a file that's in `ignores`. This flag makes it a no-op instead.
    "eslint --fix --max-warnings 0 --no-warn-ignored",
    "prettier --write"
  ],
  "*.{json,md,yml,yaml}": ["prettier --write"]
}

// ⚠️ lint-staged runs ESLint on a SUBSET of files. With type-aware rules, the
//    Program still gets built for the whole project, so a pre-commit hook can
//    take 20s+. Two mitigations:
//      a) accept it — 20s to prevent a broken main branch is a good trade;
//      b) run only non-type-aware rules pre-commit, full set in CI:
//         "eslint --fix --rulesdir ... --no-eslintrc ..."  (fiddly; usually not worth it)
```

```ts
// ── src/routes/orderRoutes.ts — what the config forces you to write ─────────
import { Router } from "express";
import type { RequestHandler } from "express";

import type { OrderService } from "../services/orderService.js";
import { logger } from "../logger.js";

// no-misused-promises makes this wrapper mandatory on Express 4.
// Without it, an async handler passed directly to app.get() is an error.
const asyncHandler =
  (fn: (...args: Parameters<RequestHandler>) => Promise<void>): RequestHandler =>
  (req, res, next) => {
    fn(req, res, next).catch(next);   // ← rejection reaches error middleware
  };

export function createOrderRoutes(orderService: OrderService): Router {
  const router = Router();

  router.get(
    "/orders/:orderId",
    asyncHandler(async (req, res) => {
      const orderId = req.params.orderId;

      // strict-boolean-expressions: `if (orderId)` would be an error, because
      // an empty string is a real (and wrong) value here, not "missing".
      if (orderId.length === 0) {
        res.status(400).json({ error: "MISSING_ORDER_ID" });
        return;
      }

      const order = await orderService.findById(orderId);

      // allowNullableObject: true means this form stays legal.
      if (order === null) {
        res.status(404).json({ error: "ORDER_NOT_FOUND" });
        return;
      }

      res.json({ data: order });
    })
  );

  router.post(
    "/orders/:orderId/cancel",
    asyncHandler(async (req, res) => {
      await orderService.cancel(req.params.orderId, "customer_request");

      // Fire-and-forget analytics: `void` is the explicit, rule-approved way
      // to say "I know this is a promise and I'm choosing not to wait".
      void analytics
        .track("order_cancelled", { orderId: req.params.orderId })
        .catch((err: unknown) => {
          logger.warn({ err }, "analytics track failed");
        });

      res.status(204).end();
    })
  );

  return router;
}
```

---

## Going deeper

### ESLint is not a formatter — stop using it as one

In April 2023 the ESLint team **deprecated all formatting rules** (`indent`, `quotes`, `semi`, `comma-dangle`, ...). typescript-eslint followed suit and moved its formatting rules out to `@stylistic/eslint-plugin-ts`.

```js
// ❌ The old way — ESLint fighting Prettier over every semicolon.
{
  rules: {
    "@typescript-eslint/indent": ["error", 2],           // deprecated
    "@typescript-eslint/quotes": ["error", "double"],    // deprecated
    "@typescript-eslint/semi": ["error", "always"],      // deprecated
    "@typescript-eslint/member-delimiter-style": "error" // deprecated
  }
}
// Symptoms: `eslint --fix` and `prettier --write` produce different output;
// running them in sequence flip-flops the file; CI is red for whitespace.

// ✅ The current way — clean separation of concerns.
//    Prettier owns:  whitespace, line breaks, quotes, semicolons, trailing commas
//    ESLint owns:    correctness, safety, consistency of CONSTRUCTS
//
// npm i -D prettier eslint-config-prettier
import eslintConfigPrettier from "eslint-config-prettier";

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  eslintConfigPrettier   // ← LAST. Turns off every rule that conflicts with Prettier.
);
```

```jsonc
// .prettierrc
{ "semi": true, "singleQuote": false, "trailingComma": "all", "printWidth": 100 }
```

Note the naming trap: `stylisticTypeChecked` is **not** about formatting. It's about *construct* consistency — `interface` vs `type`, `T[]` vs `Array<T>`, `Record<K,V>` vs an index signature. Those are semantic choices Prettier can't and shouldn't make. Keep it; it does not conflict with Prettier.

If you truly want ESLint to do formatting (e.g. you can't adopt Prettier), use `@stylistic/eslint-plugin` — the community continuation of the deprecated rules — but pick one tool, never both.

### Monorepos

```js
// ── Root eslint.config.js, shared by all packages ───────────────────────────
// projectService makes this nearly trivial: each file finds its own nearest
// tsconfig.json. With parserOptions.project you'd be maintaining a glob list.
import tseslint from "typescript-eslint";

export default tseslint.config(
  { ignores: ["**/dist/**", "**/node_modules/**", "**/coverage/**"] },
  ...tseslint.configs.strictTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname   // the REPO root
      }
    }
  },

  // Package-specific layers, keyed by path:
  {
    files: ["services/api/**/*.ts"],
    rules: { "no-console": "error" }         // services log through pino
  },
  {
    files: ["packages/cli/**/*.ts"],
    rules: { "no-console": "off" }           // the CLI's whole job is printing
  },
  {
    files: ["packages/shared/**/*.ts"],
    rules: {
      // A shared library must never reach into a consumer.
      "no-restricted-imports": ["error", {
        patterns: [{ group: ["@acme/api", "@acme/worker"],
                     message: "shared/ must not depend on services" }]
      }]
    }
  }
);
```

```jsonc
// If a package needs its own config, put eslint.config.js in that package —
// ESLint uses the nearest one to the CWD. Import the root one and extend:
// services/api/eslint.config.js
import base from "../../eslint.config.js";
export default [...base, { rules: { "no-console": "error" } }];
```

For very large monorepos, prefer running lint through your task runner so it's cached and parallelised per package:

```bash
turbo run lint            # or: nx run-many -t lint, pnpm -r lint
```

### Writing a custom rule for your codebase

Once you have type-aware linting, you can encode domain-specific invariants that no generic rule could know.

```ts
// ── tools/eslint-rules/noRawSqlOutsideRepositories.ts ───────────────────────
import { ESLintUtils, type TSESTree } from "@typescript-eslint/utils";

const createRule = ESLintUtils.RuleCreator(
  (name) => `https://internal.acme.dev/lint/${name}`
);

export const noRawSqlOutsideRepositories = createRule({
  name: "no-raw-sql-outside-repositories",
  meta: {
    type: "problem",
    docs: { description: "Raw SQL must live in src/repositories/." },
    messages: {
      rawSql: "Raw SQL found outside src/repositories/. Move this query into a repository."
    },
    schema: []
  },
  defaultOptions: [],
  create(context) {
    // Only enforce outside the repository layer.
    if (context.filename.includes("/src/repositories/")) return {};

    return {
      // Matches pool.query(...) / client.query(...) / db.query(...)
      "CallExpression > MemberExpression[property.name='query']"(
        node: TSESTree.MemberExpression
      ) {
        context.report({ node, messageId: "rawSql" });
      }
    };
  }
});
```

```js
// ── Wiring it in ────────────────────────────────────────────────────────────
import { noRawSqlOutsideRepositories } from "./tools/eslint-rules/index.js";

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  {
    plugins: { acme: { rules: { "no-raw-sql-outside-repositories": noRawSqlOutsideRepositories } } },
    rules: { "acme/no-raw-sql-outside-repositories": "error" }
  }
);
```

Type-aware version — asking the checker rather than matching syntax:

```ts
create(context) {
  const services = ESLintUtils.getParserServices(context);   // ← needs projectService
  const checker = services.program.getTypeChecker();

  return {
    AwaitExpression(node) {
      const type = checker.getTypeAtLocation(services.esTreeNodeToTSNodeMap.get(node.argument));
      // Now you can reason about the ACTUAL type, not the source text:
      // is it a Promise? does it have a `.rows` property? is it our Result<T,E>?
    }
  };
}
```

### Adopting lint on an existing codebase without a 2,000-error PR

```js
// ── Strategy 1: warn now, error later ───────────────────────────────────────
{
  rules: {
    "@typescript-eslint/no-unsafe-assignment": "warn",   // 400 findings
    "@typescript-eslint/no-explicit-any": "warn"         // 180 findings
  }
}
// CI runs `eslint .` (warnings don't fail) but NOT --max-warnings 0.
// Track the count down over sprints, then flip to "error" and add the flag.

// ── Strategy 2: strict for new code only ────────────────────────────────────
{
  files: ["src/modules/billing/**/*.ts", "src/modules/notifications/**/*.ts"],
  extends: [tseslint.configs.strictTypeChecked]   // new modules get the full set
}
// Everything else stays on recommendedTypeChecked. Move modules over one at a time.

// ── Strategy 3: lint only changed files in CI ───────────────────────────────
// .github/workflows/ci.yml
//   - run: |
//       CHANGED=$(git diff --name-only --diff-filter=ACMR origin/main...HEAD \
//                 -- '*.ts' | tr '\n' ' ')
//       if [ -n "$CHANGED" ]; then npx eslint $CHANGED --max-warnings 0; fi
//
// ⚠️ Incomplete by construction: a change in fileA.ts can introduce a
//    type-aware violation in fileB.ts. Use as a transition step, then run the
//    full lint as a nightly job until you can make it blocking on every PR.

// ── Strategy 4: a suppressions baseline ─────────────────────────────────────
// ESLint 9.24+ ships this natively:
npx eslint --suppress-all                  // writes eslint-suppressions.json
npx eslint                                 // existing violations suppressed;
                                           // NEW ones still fail
npx eslint --prune-suppressions            // drop entries you've since fixed
// This is the cleanest option: main goes green immediately, the debt is
// itemised in a file you can burn down, and no new debt can be added.
```

### The rules people always end up turning off (and whether they should)

```ts
// ── @typescript-eslint/no-non-null-assertion ────────────────────────────────
// Fires on every `!`. Very common to disable.
// VERDICT: keep it on in src/, off in tests. If you need `!` in src/, you
// usually needed an assertion function instead (see doc 63):
function assertOrderExists(o: Order | null): asserts o is Order {
  if (o === null) throw new OrderNotFoundError();
}

// ── @typescript-eslint/explicit-module-boundary-types ───────────────────────
// Demands a return type on every exported function.
// VERDICT: off. TypeScript's inference is excellent and the annotations rot.
// Exception: turn it ON for a published library's public API, where an
// accidentally-widened inferred return type is a breaking change.

// ── @typescript-eslint/no-unsafe-assignment (and friends) ───────────────────
// Fires constantly if you use untyped deps.
// VERDICT: keep on. Then fix the root cause — write a .d.ts (doc 49), or wrap
// the untyped module in ONE typed adapter file where you disable it locally.

// ── @typescript-eslint/strict-boolean-expressions ───────────────────────────
// The most-disabled rule in this document.
// VERDICT: enable with allowNullableObject: true. That single option removes
// ~80% of the noise while keeping the "" and 0 bug detection, which is the part
// that actually catches production defects.

// ── @typescript-eslint/no-unnecessary-condition ─────────────────────────────
// Fires at every trust boundary where reality may not match the declared type.
// VERDICT: keep on, and treat each finding as a question: "is this type a lie?"
// If yes, add runtime validation (zod). If no, delete the check.

// ── @typescript-eslint/no-misused-promises with checksVoidReturn ────────────
// Fires on every async callback passed to a sync API.
// VERDICT: never disable wholesale. If a specific library is the problem,
// scope it:
{
  files: ["src/queue/**/*.ts"],
  rules: { "@typescript-eslint/no-misused-promises":
             ["error", { checksVoidReturn: { arguments: false } }] }
}
```

### Debugging the config itself

```bash
# Which config objects apply to this file? Opens an interactive UI.
npx eslint --inspect-config

# Print the fully-resolved config for one file as JSON.
npx eslint --print-config src/services/orderService.ts

# Which rule produced this message?
npx eslint . --format=json | jq '.[].messages[].ruleId' | sort | uniq -c | sort -rn

# Timing: which rules are slow? (type-aware ones dominate)
TIMING=10 npx eslint .
# Rule                                    | Time (ms) | Relative
# @typescript-eslint/no-unnecessary-condition | 4821.3 |    31.2%
# @typescript-eslint/no-floating-promises     | 2104.7 |    13.6%

# Is the slowness the Program build or the rules?
npx eslint . --stats --format=json | jq '.[].stats.times'

# Full debug trace (very verbose, but shows tsconfig resolution).
DEBUG=eslint:*,typescript-eslint:* npx eslint src/server.ts
```

### The `unbound-method` rule and class-based services

```ts
// A rule from recommendedTypeChecked that confuses everyone the first time.
class OrderService {
  constructor(private readonly repo: OrderRepository) {}
  async findById(orderId: string) { return this.repo.findById(orderId); }
}

const service = new OrderService(repo);

// ❌ unbound-method: `findById` is extracted from its object. `this` is lost.
//    At runtime: TypeError: Cannot read properties of undefined (reading 'repo')
const ids = orderIds.map(service.findById);

// ✅ three fixes:
const ids = orderIds.map((id) => service.findById(id));   // arrow — preferred
const ids = orderIds.map(service.findById.bind(service));
// or define the method as an arrow property so `this` is captured:
class OrderService {
  findById = async (orderId: string) => this.repo.findById(orderId);
}
```

---

## Common mistakes

### Mistake 1 — Expecting `tsc` to catch lint-class bugs, so never adding a linter

```jsonc
// ❌ "We have strict TypeScript, we don't need ESLint."
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

```ts
// This all compiles cleanly under the config above, and every line is a bug:
async function processRefund(orderId: string): Promise<void> {
  notifyAccounting(orderId);                 // floating → unhandled rejection → crash
  const amount: any = await fetchAmount();   // any → all downstream safety gone
  if (amount.cents) { /* 0 refund silently skipped */ }
}
app.post("/refunds", async (req, res) => { /* rejection escapes Express */ });
```

```bash
# ✅ Both. Always both. They check disjoint things.
npm run typecheck    # tsc --noEmit
npm run lint:ci      # eslint . --max-warnings 0
```

### Mistake 2 — Enabling type-aware presets but never configuring the parser

```js
// ❌ The preset is loaded, but the rules in it silently do nothing —
//    or crash with "you have used a rule which requires type information".
export default tseslint.config(
  ...tseslint.configs.recommendedTypeChecked
  // ← no languageOptions.parserOptions at all
);

// Error you'll see:
//   Error while loading rule '@typescript-eslint/no-floating-promises':
//   You have used a rule which requires type information, but don't have
//   parserOptions set to generate type information for this file.
```

```js
// ✅ The parser must be told how to build a Program.
export default tseslint.config(
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname   // ← without this, relative
                                               //   resolution depends on CWD and
                                               //   breaks when CI runs from elsewhere
      }
    }
  }
);
```

### Mistake 3 — Leaving type-aware rules on for files with no TypeScript project

```js
// ❌ eslint.config.js, vitest.config.ts, and scripts/*.ts aren't in tsconfig's
//    `include`, so every one of them fails to parse:
//
//    error  Parsing error: ESLint was configured to run on
//    `<tsconfigRootDir>/vitest.config.ts` using `parserOptions.project`.
//    However, that TSConfig does not include this file.
export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  { languageOptions: { parserOptions: { project: "./tsconfig.json" } } }
);
```

```js
// ✅ Three fixes, best first.

// (a) Add them to a tsconfig's include — they ARE part of your project:
//     "include": ["src/**/*.ts", "test/**/*.ts", "*.config.ts", "scripts/**/*.ts"]

// (b) Use projectService with an escape hatch for the few stragglers:
{
  languageOptions: {
    parserOptions: {
      projectService: { allowDefaultProject: ["*.config.ts"] },
      tsconfigRootDir: import.meta.dirname
    }
  }
}

// (c) Turn the type-aware rules off for those files. Must come LAST.
{
  files: ["**/*.js", "*.config.ts"],
  extends: [tseslint.configs.disableTypeChecked]
}
```

### Mistake 4 — Disabling `no-floating-promises` because it "fires everywhere"

```ts
// ❌ 60 errors on day one, so:
{ rules: { "@typescript-eslint/no-floating-promises": "off" } }
// You have just turned off the rule that prevents your Node process from
// exiting on an unhandled rejection. Those 60 findings were 60 real risks.
```

```ts
// ✅ Triage them. There are only ever four correct answers:

// 1. It should have been awaited — the common case, and a real bug.
await orderRepository.markCancelled(orderId);

// 2. It's genuinely fire-and-forget, and you accept losing errors.
//    `void` makes that intent explicit and greppable.
void analytics.track("order_cancelled", { orderId });

// 3. It's fire-and-forget but errors MUST be logged (usually the right answer).
notificationService
  .send(orderId)
  .catch((err: unknown) => { logger.error({ err, orderId }, "notify failed"); });

// 4. You're intentionally starting work in parallel — collect it later.
const pending = [sendEmail(orderId), sendSms(orderId), writeAudit(orderId)];
const results = await Promise.allSettled(pending);

// If it's genuinely noisy for one library, scope it — never globally:
{
  files: ["src/telemetry/**/*.ts"],
  rules: { "@typescript-eslint/no-floating-promises": "off" }
}
```

### Mistake 5 — Turning off `@typescript-eslint/no-unused-vars` because of `_req`

```ts
// ❌ Express middleware signatures require positional params you don't use:
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  res.status(500).json({ error: "INTERNAL" });
});
// → 'req' is defined but never used.  So someone does:
{ rules: { "@typescript-eslint/no-unused-vars": "off" } }
// Now genuinely dead imports and variables accumulate forever.
```

```ts
// ✅ Configure the ignore patterns instead. Underscore-prefix is the convention.
{
  rules: {
    "no-unused-vars": "off",                        // ← the core rule MUST be off
    "@typescript-eslint/no-unused-vars": ["error", {
      args: "after-used",                    // only flag unused args AFTER the last used one
      argsIgnorePattern: "^_",               // _req, _next
      varsIgnorePattern: "^_",
      caughtErrorsIgnorePattern: "^_",       // catch (_err)
      destructuredArrayIgnorePattern: "^_",  // const [, _second] = pair
      ignoreRestSiblings: true               // const { password, ...safe } = user
    }]
  }
}
```

### Mistake 6 — Running ESLint and Prettier as competing formatters

```js
// ❌ Both tools claim ownership of quotes and semicolons.
export default tseslint.config(
  ...tseslint.configs.stylisticTypeChecked,
  {
    rules: {
      "@typescript-eslint/quotes": ["error", "single"],   // deprecated + conflicts
      "@typescript-eslint/semi": ["error", "never"]       // deprecated + conflicts
    }
  }
);
// .prettierrc: { "singleQuote": false, "semi": true }
//
// Result: `npm run lint:fix && npm run format` produces a diff every time.
// CI fails on whitespace. Everyone loses an hour.
```

```js
// ✅ Prettier owns formatting; eslint-config-prettier disables the conflicts.
import eslintConfigPrettier from "eslint-config-prettier";

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  eslintConfigPrettier            // ← MUST be last: it only turns rules OFF
);
```

### Mistake 7 — Not failing CI on warnings

```yaml
# ❌ Warnings accumulate to 400 and nobody looks at the log any more.
- run: npx eslint .
  # exit code 0 as long as there are no ERRORS. Warnings are invisible in CI.
```

```yaml
# ✅ Either make everything an error, or make warnings fail.
- run: npx eslint . --max-warnings 0
```

```js
// And forbid the other slow leak — stale disable comments that suppress
// violations introduced years after the comment was written:
{ linterOptions: { reportUnusedDisableDirectives: "error" } }
```

### Mistake 8 — Trusting `--cache` in CI with type-aware rules

```yaml
# ❌ Restoring .eslintcache across CI runs with type-aware linting.
- uses: actions/cache@v4
  with: { path: .eslintcache, key: eslint-${{ github.sha }} }
- run: npx eslint . --cache --max-warnings 0
#
# The cache key is per-file (mtime/content). But no-floating-promises' verdict
# on fileA.ts depends on the RETURN TYPE declared in fileB.ts. Change fileB,
# and fileA is served from cache — the new violation is never reported.
```

```yaml
# ✅ Cache node_modules, not lint results. Lint runs from scratch.
- uses: actions/setup-node@v4
  with: { node-version: "22", cache: "npm" }
- run: npm ci
- run: npx eslint . --max-warnings 0
# Use --cache locally, where a false negative costs you nothing and CI catches it.
```

---

## Practice exercises

### Exercise 1 — easy

Stand up a working type-aware ESLint setup from zero and observe each rule fire.

Concretely:

1. `mkdir eslint-ts-lab && cd eslint-ts-lab && npm init -y && npm pkg set type=module`
2. `npm i -D eslint typescript typescript-eslint @types/node`
3. Write `tsconfig.json` with `strict: true`, `target: "ES2022"`, `module: "NodeNext"`, `rootDir: "src"`, `outDir: "dist"`, and `include: ["src/**/*.ts", "eslint.config.js"]`.
4. Write `eslint.config.js` as a flat config using `tseslint.config()`, composing `js.configs.recommended` + `tseslint.configs.recommendedTypeChecked`, with `projectService: true` and `tsconfigRootDir: import.meta.dirname`.
5. Write `src/inventoryService.ts` containing **one deliberate violation of each** of: `no-floating-promises`, `no-explicit-any`, `consistent-type-imports`, and `no-unused-vars`. Use real names — `restockWarehouseItem`, `sku`, `quantityOnHand`.
6. Run `npx eslint .` and record the exact rule ID and message for each of the four.
7. Run `npx eslint . --fix`. Note precisely which two are auto-fixed and which two are not, and write one sentence explaining why a floating promise cannot be auto-fixed.
8. Now **remove** `projectService` from the config and re-run. Record the exact error message you get, then put it back.
9. Add `npm run lint` and `npm run typecheck` scripts, and confirm that `tsc --noEmit` reports **zero** errors on a file where ESLint reports three.

```
// Write your code here
```

### Exercise 2 — medium

Build the lint configuration for a realistic Express + Postgres service, with tests and CI.

Concretely:

1. Scaffold: `src/server.ts`, `src/config.ts`, `src/routes/paymentRoutes.ts`, `src/services/paymentService.ts`, `src/repositories/paymentRepository.ts`, `test/paymentService.test.ts`, `scripts/reconcilePayments.ts`, `vitest.config.ts`.
2. Write `tsconfig.json` (src only) and `tsconfig.test.json` (src + test). Confirm both are discovered by `projectService` with no glob list.
3. Compose `strictTypeChecked` + `stylisticTypeChecked` + `eslint-config-prettier`. Verify with `npx eslint --print-config src/services/paymentService.ts` that Prettier-conflicting rules are `"off"`.
4. Explicitly enable and configure all seven backend-critical rules: `no-floating-promises` (with `checkThenables: true`), `no-misused-promises` (with the full `checksVoidReturn` object), `consistent-type-imports` (with `fixStyle: "inline-type-imports"`), `no-explicit-any`, `no-unnecessary-condition`, `strict-boolean-expressions` (with `allowNullableObject: true`), and `require-await`.
5. Write an async Express route handler passed **directly** to `router.post()`. Confirm `no-misused-promises` errors. Then write an `asyncHandler` wrapper that forwards rejections to `next` and confirm the error clears.
6. Write code that trips `strict-boolean-expressions` in three distinct ways: a truthy check on a `string`, on a `number`, and on a `string | undefined`. Fix each one explicitly.
7. Add a per-file override for `test/**` and `**/*.test.ts` that disables the `no-unsafe-*` family and `no-non-null-assertion`, but **keeps** `no-floating-promises` and `no-misused-promises`. Write a comment in the config explaining why those two stay on in tests.
8. Add an override for `scripts/**` and `*.config.ts` using `tseslint.configs.disableTypeChecked`, plus `no-console: "off"`.
9. Add a global `ignores` entry for `dist/**`, `coverage/**`, and `src/generated/**`.
10. Set `linterOptions.reportUnusedDisableDirectives: "error"`. Add an `eslint-disable-next-line` for a rule that is not actually violated and confirm ESLint now fails on the useless comment.
11. Write `.github/workflows/ci.yml` with `typecheck` and `lint` as separate parallel jobs. Lint must run `--max-warnings 0` and set `NODE_OPTIONS=--max-old-space-size=6144`.
12. Measure: run `TIMING=10 npx eslint .` and record which three rules are slowest. Then delete `projectService` and re-measure the total wall-clock time. Write down both numbers and the ratio.

```
// Write your code here
```

### Exercise 3 — hard

Take a deliberately messy codebase to a fully green, strict, type-aware lint gate — and write your own rule.

Concretely:

1. **Create the mess.** Write a `src/` tree of at least 12 files for an order-management service (routes, services, repositories, a webhook handler, a queue worker, migrations). Seed it with at least 40 violations spanning: floating promises, async handlers in sync positions, `any` from `JSON.parse` and an untyped SDK, truthy checks on strings and numbers, dead conditions after a type was tightened, `async` functions with no `await`, runtime imports of type-only modules, `throw "string"`, and unbound methods passed to `.map()`.
2. **Baseline.** Turn on `strictTypeChecked` + `stylisticTypeChecked` and capture the full error count per rule: `npx eslint . --format=json | jq -r '.[].messages[].ruleId' | sort | uniq -c | sort -rn`. Save it as `lint-baseline.txt`.
3. **Suppress and burn down.** Run `npx eslint --suppress-all` to generate `eslint-suppressions.json` and verify CI is green. Then fix violations in three passes, running `npx eslint --prune-suppressions` after each, and record the count after each pass. Explain in two sentences why this is better than setting the rules to `"warn"`.
4. **Fix the `any` at the source.** Replace `JSON.parse` in the webhook handler with a validated parse (zod or a hand-written type guard from doc 41) so that `no-unsafe-assignment`, `no-unsafe-member-access`, and `no-unsafe-argument` all clear without a single disable comment. For the untyped SDK, write a `.d.ts` (doc 49) instead of disabling rules.
5. **Prove the async rules matter.** Write a failing integration test that demonstrates the un-awaited promise causing an unhandled rejection. Fix the code, and show the test now passes. Do the same for an async `.forEach` losing writes.
6. **Write a custom type-aware rule.** Using `ESLintUtils.RuleCreator` and `getParserServices()`, implement `acme/no-raw-pool-query-outside-repositories`: it must report any call to a method named `query` on an object whose type comes from `pg`, unless the file path is under `src/repositories/`. Include `meta.messages`, `meta.docs`, and a `schema`. Write unit tests for it with `@typescript-eslint/rule-tester` covering at least two valid and three invalid cases.
7. **Monorepo split.** Restructure into `packages/shared`, `services/api`, `services/worker`. Write a single root `eslint.config.js` where `projectService` resolves all three packages' tsconfigs with no explicit project list. Add a `no-restricted-imports` rule preventing `packages/shared` from importing anything under `services/`, and prove it fires.
8. **Performance.** Measure and report: (a) full type-aware lint, cold; (b) full type-aware lint with `--cache`, warm; (c) `disableTypeChecked` everywhere. Then construct a concrete case where the `--cache` result is *wrong*: change a return type in `packages/shared` so it introduces a `no-floating-promises` violation in `services/api`, and show that the cached run misses it.
9. **Ship the gate.** Write the final CI workflow with parallel `typecheck` / `lint` / `test` jobs, SARIF upload for inline PR annotations, and a `husky` + `lint-staged` pre-commit hook. Add a `CONTRIBUTING.md` section documenting: which rules are non-negotiable and why, the approved format for a disable comment (with `--` reason), and the escalation path for proposing a rule change.

```
// Write your code here
```

---

## Quick reference cheat sheet

```bash
# ── Install ──────────────────────────────────────────────────────────────────
npm i -D eslint typescript typescript-eslint
npm i -D prettier eslint-config-prettier          # if using Prettier
npm i -D globals                                   # for languageOptions.globals

# ── Run ──────────────────────────────────────────────────────────────────────
npx eslint .                          # lint everything flat config matches
npx eslint . --fix                    # auto-fix
npx eslint . --max-warnings 0         # CI: warnings are failures
npx eslint . --cache                  # local speedup (unsafe as a sole CI gate)
npx eslint --inspect-config           # interactive: which config applies where
npx eslint --print-config src/x.ts    # resolved config for one file
npx eslint --suppress-all             # baseline existing violations
npx eslint --prune-suppressions       # drop suppressions you've fixed
TIMING=10 npx eslint .                # per-rule timings
DEBUG=eslint:* npx eslint src/x.ts    # verbose trace
npx @eslint/migrate-config .eslintrc.json   # .eslintrc → flat config
```

```js
// ── Minimal flat config ──────────────────────────────────────────────────────
import js from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  { ignores: ["dist/**", "coverage/**"] },
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname }
    },
    linterOptions: { reportUnusedDisableDirectives: "error" }
  },
  { files: ["**/*.js"], extends: [tseslint.configs.disableTypeChecked] }
);
```

| Preset | Type info? | Rules | Use when |
|---|---|---|---|
| `recommended` | no | ~35 | Fast lint, legacy migration |
| `recommendedTypeChecked` | **yes** | ~55 | Sensible default for a service |
| `strict` | no | ~65 | Fast lint, opinionated |
| `strictTypeChecked` | **yes** | ~95 | New codebase, max safety |
| `stylistic` | no | ~20 | Compose with any of the above |
| `stylisticTypeChecked` | **yes** | ~25 | Compose with any of the above |
| `disableTypeChecked` | — | — | Last, for `.js` / config files |

| Backend-critical rule | Type-aware | Catches |
|---|---|---|
| `no-floating-promises` | ✅ | Unhandled rejection → process crash |
| `no-misused-promises` | ✅ | Async Express handler, async `.filter`/`.forEach` |
| `await-thenable` | ✅ | `await` on a non-promise |
| `require-await` | ✅ | `async` with no `await` |
| `return-await` | ✅ | Missing `return await` inside `try` |
| `consistent-type-imports` | ❌ | Runtime import of type-only module |
| `no-explicit-any` | ❌ | Written `any` |
| `no-unsafe-assignment` / `-member-access` / `-call` / `-return` / `-argument` | ✅ | `any` leaking in from untyped deps |
| `no-unnecessary-condition` | ✅ | Dead guards after a type was tightened |
| `strict-boolean-expressions` | ✅ | `""` and `0` treated as "missing" |
| `restrict-template-expressions` | ✅ | `${obj}` → `"[object Object]"` |
| `only-throw-error` | ✅ | `throw "message"` |
| `unbound-method` | ✅ | Method passed without `this` |
| `no-unnecessary-type-assertion` | ✅ | Redundant `as X` |

```ts
// ── Disable comments ─────────────────────────────────────────────────────────
// eslint-disable-next-line <rule> -- reason      ← preferred form
const x = y;  // eslint-disable-line <rule> -- reason
/* eslint-disable <rule> */ ... /* eslint-enable <rule> */
// @ts-expect-error -- reason                      ← for tsc, not ESLint

// ── Extension rules: ALWAYS turn the core one off ────────────────────────────
"no-unused-vars": "off",       "@typescript-eslint/no-unused-vars": "error"
"require-await": "off",        "@typescript-eslint/require-await": "error"
"no-shadow": "off",            "@typescript-eslint/no-shadow": "error"
"dot-notation": "off",         "@typescript-eslint/dot-notation": "error"
"no-use-before-define": "off", "@typescript-eslint/no-use-before-define": "error"
```

| Symptom | Cause | Fix |
|---|---|---|
| `Parsing error: Unexpected token :` | espree parsing TS | Use `typescript-eslint` |
| `rule which requires type information` | No `projectService`/`project` | Add `parserOptions.projectService: true` |
| `TSConfig does not include this file` | File outside `include` | Add to `include`, or `allowDefaultProject`, or `disableTypeChecked` |
| Lint takes 90s | Type-aware Program build | Expected; parallelise in CI, `--cache` locally |
| `JavaScript heap out of memory` | Big Program | `NODE_OPTIONS=--max-old-space-size=8192` |
| ESLint and Prettier fight | Formatting rules on | Add `eslint-config-prettier` last |
| `no-undef` false positives on TS | Core rule can't see types | Turn it off; `tsc` already does this |
| Warnings ignored in CI | Exit code 0 | `--max-warnings 0` |
| `.eslintignore` has no effect | Flat config ignores it | Use `{ ignores: [...] }` |
| Overrides not applying | Object order / glob | Put overrides last; use `**/*.x` not `*.x` |
| Stale `eslint-disable` hides a bug | No reporting | `reportUnusedDisableDirectives: "error"` |
| `Cannot use import statement` in config | CJS package | `"type": "module"` or `eslint.config.mjs` |

---

## Connected topics

- **03 — tsconfig in depth** — `strict`, `strictNullChecks`, and `verbatimModuleSyntax` are prerequisites: `no-unnecessary-condition` and `strict-boolean-expressions` refuse to run without `strictNullChecks`, and `consistent-type-imports` only becomes load-bearing under `verbatimModuleSyntax`.
- **04 — TypeScript Node.js project setup** — where the `lint`, `typecheck`, and `check` npm scripts live, and how they fit the dev loop.
- **07 — Special types** — `any` vs `unknown` vs `never`: the distinction `no-explicit-any` and the `no-unsafe-*` family exist to enforce.
- **41 — Type guards** and **63 — Assertion functions** — the correct alternatives to the `!` that `no-non-null-assertion` bans.
- **49 — Declaration files** — writing a `.d.ts` for an untyped dependency is the real fix for `no-unsafe-*` errors, not a disable comment.
- **60 — Error handling in TypeScript** — `only-throw-error`, `use-unknown-in-catch-callback-variable`, and `prefer-promise-reject-errors` encode the practices from that document.
- **62 — Strict null checks** — the compiler setting that unlocks half the type-aware rules here.
- **69 — Enum vs union types** — the `no-restricted-syntax` rule banning `TSEnumDeclaration` in Example 2 comes from that argument.
- **75 — Debugging TypeScript in VS Code** — the other half of the local tooling story; both rely on the parser understanding your `tsconfig.json`.
