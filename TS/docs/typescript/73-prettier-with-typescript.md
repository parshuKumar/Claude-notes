# 73 — Prettier with TypeScript

## What is this?

**Prettier** is an opinionated code formatter. You give it a file; it throws away almost everything about how you wrote the code — line breaks, indentation, quote style, spacing, where the closing paren went — and reprints the file from scratch from its own internal representation.

That last sentence is the whole product. Prettier does not "tidy" your code. It **parses it into an AST, discards the original formatting, and pretty-prints the AST**.

```
  src/services/orderService.ts        ← what you typed, however you typed it
            │
            │  1. parse with @typescript-eslint-independent TS parser
            ▼
        AST (Abstract Syntax Tree)    ← formatting is GONE at this point
            │
            │  2. print with a line-width-aware layout algorithm
            ▼
  src/services/orderService.ts        ← deterministic output, byte-identical for
                                        every developer on the team
```

Two consequences follow directly from that pipeline, and they explain roughly every question people have about Prettier:

1. **If it can't parse, it can't format.** A syntax error means Prettier fails loudly and changes nothing. This is a feature: Prettier can never corrupt broken code.
2. **Your original formatting carries no information** — with exactly one exception, the blank line. Prettier preserves whether *a* blank line existed between two statements (collapsing 2+ to 1), because that is the one whitespace signal humans genuinely use to group code.

Prettier supports TypeScript natively. There is no plugin to install, no `parser` to configure in the common case — `.ts` and `.tsx` files are first-class, and the bundled `typescript` parser understands generics, decorators, `satisfies`, `const` type parameters, and everything else the language has shipped.

---

## Why does it matter?

Because formatting arguments are the single highest-volume, lowest-value discussion in a code review, and Prettier deletes the entire category.

Concretely, on a backend team:

- **Diffs stop lying.** Without a formatter, a PR that changes one line of `orderService.ts` also reindents 40 lines because the author's editor uses 4 spaces. `git blame` becomes useless. With Prettier, the diff is one line.
- **Review comments become about behaviour.** No more "can you break this argument list?" — the tool already did.
- **Onboarding is a `npm install`.** A new hire doesn't need to absorb an unwritten style guide; they save the file and it conforms.
- **Merge conflicts drop.** Two people touching the same function no longer conflict on whitespace.

TypeScript specifically raises the stakes, because TS code has more *long* constructs than JS: generic signatures, union types that don't fit on one line, `Promise<Result<UserRecord, DatabaseError>>` return annotations, decorator stacks in NestJS. These are exactly the constructs where humans disagree most about line breaking, and exactly where a deterministic printer earns its keep.

```ts
// Written by three different people, all "correct":
export async function findUsersByTenant(tenantId: string, options: { limit?: number; offset?: number; includeDeleted?: boolean }): Promise<UserRecord[]> {}

export async function findUsersByTenant(
  tenantId: string,
  options: { limit?: number, offset?: number, includeDeleted?: boolean }
): Promise<UserRecord[]> {}

export async function findUsersByTenant( tenantId : string ,
    options : { limit? : number ; offset? : number ; includeDeleted? : boolean } )
        : Promise <UserRecord[]> {}

// After Prettier — all three become byte-identical:
export async function findUsersByTenant(
  tenantId: string,
  options: { limit?: number; offset?: number; includeDeleted?: boolean },
): Promise<UserRecord[]> {}
```

And the second, quieter reason: **Prettier is a precondition for useful linting.** Once formatting is mechanical, your ESLint config can be 100% about correctness rules (`no-floating-promises`, `no-explicit-any`, exhaustiveness) instead of half stylistic noise. That separation is the subject of a whole section below.

---

## The JavaScript way vs the TypeScript way

The honest headline: **Prettier works identically on JS and TS.** There is no separate TypeScript mode you enable. But the surrounding setup differs in three real ways.

```js
// ── The JavaScript way — Prettier on a plain Node project ────────────────────

// npm i -D prettier
//
// .prettierrc
// { "semi": true, "singleQuote": true }
//
// That's it. Prettier detects .js / .mjs / .cjs and uses the "babel" parser.
// Your ESLint config probably also had stylistic rules (indent, quotes, semi)
// from eslint:recommended-era configs, which now fight Prettier — you install
// eslint-config-prettier to switch those off.

// The formatting surface is small: statements, objects, arrays, calls, JSX.
const createUser = async (email, name) => {
  const user = await db.users.insert({ email, name });
  return user;
};
```

```ts
// ── The TypeScript way — same tool, three additional considerations ──────────

// 1. PARSER. Prettier auto-selects the "typescript" parser for .ts/.tsx/.mts/.cts.
//    You almost never set `parser` by hand. The exception is formatting TS
//    embedded in an unusual extension, where you use an `overrides` block.

// 2. LARGER FORMATTING SURFACE. TS adds constructs that only exist here, and
//    Prettier has specific opinions about each:
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";  // unions
interface Repo<T extends { id: string }> {                        // generics
  findById(id: string): Promise<T | null>;
}
@Injectable()                                                     // decorators
export class OrderService implements Repo<Order> {
  constructor(private readonly db: Database) {}                   // param properties
  async findById(id: string): Promise<Order | null> {
    return (await this.db.query(sql, [id])) satisfies Order | null; // satisfies
  }
}

// 3. NEW CONFIG KEYS BECOME LOAD-BEARING.
//    - trailingComma affects generic parameter lists and type members
//    - semi interacts with TS's ASI hazards around type-only syntax
//    - arrowParens changes nothing semantically but shows up in every callback
//
//    And ONE genuinely TS-only key you will meet in older configs:
//    "parser": "typescript"  — required in Prettier 1.x, auto-detected since 2.x.
```

The revelation for a JS developer: **nothing you know changes.** Prettier's TypeScript support is not an add-on you must configure; it is built into the core package. The work is not "make Prettier understand TS" — it is "stop ESLint from arguing with Prettier, and put Prettier in CI so it's enforced rather than suggested."

---

## Syntax

```bash
# ── Install ──────────────────────────────────────────────────────────────────
npm i -D --save-exact prettier
#         ^^^^^^^^^^^ pin the EXACT version. A minor Prettier bump can change
#                     output (they do this deliberately), and an unpinned range
#                     means CI reformats the world on a random Tuesday.

# ── The three commands you will actually use ─────────────────────────────────
npx prettier --write .                # format everything, in place
npx prettier --check .                # exit 1 if anything is unformatted — for CI
npx prettier --write src/services      # format a subtree
npx prettier --check "src/**/*.{ts,tsx}"  # quote globs so your shell doesn't expand them

# ── Diagnostics ──────────────────────────────────────────────────────────────
npx prettier --find-config-path src/server.ts   # WHICH config file applies here?
npx prettier --file-info src/server.ts          # parser chosen + ignored?
npx prettier --support-info                     # every option, every language
npx prettier --write . --log-level debug        # why is this file being skipped?
```

```jsonc
// ── .prettierrc — the options that actually matter for TypeScript ────────────
// (JSON with comments is fine in .prettierrc; use .prettierrc.json for strict JSON)
{
  "semi": true,              // end statements with ; — see the big argument below
  "singleQuote": false,      // "double" quotes. Prettier's default.
  "trailingComma": "all",    // trailing commas everywhere, incl. function params
  "printWidth": 100,         // target line length. NOT a hard limit — a preference.
  "tabWidth": 2,             // spaces per indent level
  "useTabs": false,          // spaces, not tabs (tabs are better for a11y; team call)
  "arrowParens": "always",   // (x) => x   instead of   x => x
  "bracketSpacing": true,    // { foo: 1 }  instead of  {foo: 1}
  "bracketSameLine": false,  // JSX: closing > on its own line
  "quoteProps": "as-needed", // only quote object keys that require it
  "endOfLine": "lf",         // NORMALISE line endings — critical on mixed Win/mac teams
  "jsxSingleQuote": false,   // JSX attributes use "double"
  "singleAttributePerLine": false,
  "embeddedLanguageFormatting": "auto",  // format CSS/GraphQL/HTML inside template literals

  // Per-glob overrides — the escape hatch for "everything except these files"
  "overrides": [
    {
      "files": ["*.md", "*.mdx"],
      "options": { "printWidth": 80, "proseWrap": "preserve" }
    },
    {
      "files": ["*.json", ".prettierrc"],
      "options": { "parser": "json", "trailingComma": "none" }
    },
    {
      "files": ["package.json"],
      "options": { "useTabs": false, "tabWidth": 2 }
    }
  ]
}
```

```
# ── .prettierignore — same syntax as .gitignore ──────────────────────────────
dist/                    # build output — formatting it is pure churn
build/
coverage/
node_modules/            # ignored by default, but be explicit
src/generated/           # the GENERATOR owns its formatting, not you
*.gen.ts
**/__snapshots__/        # reformatting invalidates every snapshot
package-lock.json        # never reformat lockfiles
pnpm-lock.yaml
!src/generated/handwritten-overrides.ts   # negation works, same as .gitignore
```

```jsonc
// ── .vscode/settings.json — format on save, committed to the repo ────────────
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",   // extension ID, not a name
  "editor.formatOnSave": true,
  // Per-language pins, because other extensions register as formatters too:
  "[typescript]":      { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[typescriptreact]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[json]":            { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "prettier.requireConfig": true,   // don't format repos that haven't opted in
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"   // run eslint --fix too, AFTER formatting
  }
}
// .vscode/extensions.json — so teammates get prompted to install it:
// { "recommendations": ["esbenp.prettier-vscode", "dbaeumer.vscode-eslint"] }
```

```json
// ── package.json scripts ─────────────────────────────────────────────────────
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint . --max-warnings=0",
    "typecheck": "tsc --noEmit",
    "verify": "npm run format:check && npm run lint && npm run typecheck"
  }
}
```

---

## How it works — concept by concept

### Concept 1 — What Prettier does, and what it emphatically does not

The line between "formatting" and "everything else" is the most important thing to internalise. Prettier owns **whitespace and delimiters**. It owns nothing else.

```ts
// ═══ WHAT PRETTIER DOES ══════════════════════════════════════════════════════

// 1. Line breaking — decides where a construct wraps, based on printWidth
const result = await orderService.createOrder(tenantId, customerId, lineItems, { idempotencyKey });
// becomes (at printWidth 80):
const result = await orderService.createOrder(
  tenantId,
  customerId,
  lineItems,
  { idempotencyKey },
);

// 2. Indentation — always derived from nesting depth, never from what you typed
// 3. Quote style — normalises ' vs " per the singleQuote option
// 4. Semicolons — adds or removes them per the semi option
// 5. Trailing commas — adds or removes per trailingComma
// 6. Spacing inside braces, around operators, after commas
// 7. Parenthesisation for clarity in a few ambiguous spots
// 8. Blank line COLLAPSING — 5 blank lines become 1; 0 stays 0

// ═══ WHAT PRETTIER DOES NOT DO ═══════════════════════════════════════════════

// ❌ It does not sort or organise imports.
//    This surprises everyone. See Concept 8.
import { z } from "zod";
import { createOrder } from "./services/orderService.js";
import express from "express";
// ↑ Prettier leaves this order exactly as-is, forever.

// ❌ It does not remove unused variables, imports, or parameters.
import { unusedHelper } from "./utils.js";   // still there after formatting

// ❌ It does not rename anything, or change any identifier.

// ❌ It does not enforce naming conventions (camelCase vs snake_case).

// ❌ It does not catch bugs. None. Zero.
const total = items.reduce((sum, i) => sum + i.price, 0);   // missing * quantity
// Prettier formats this beautifully and says nothing.

// ❌ It does not add or remove types, or touch type correctness in any way.
//    It never runs the TypeScript type checker. It only PARSES.
const userId: string = 42;   // type error; Prettier formats it and is silent.

// ❌ It does not convert `var` to `const`, or `function` to arrow, or any other
//    code transform. It is not a codemod.

// ❌ It does not enforce a maximum line length. printWidth is a *soft target*.
//    A single long string literal or a long identifier chain will exceed it,
//    and Prettier will not break the string for you.
const message = "this is a very long string literal that Prettier will not split no matter what";
```

The mental model that keeps you out of trouble:

```
Prettier   →  How the code LOOKS      (whitespace, delimiters)
ESLint     →  What the code MEANS     (bugs, patterns, correctness)
tsc        →  Whether the code is TYPE-CORRECT
```

Three tools, three jobs, no overlap. Almost every Prettier-related pain in a real repo comes from letting these responsibilities blur.

### Concept 2 — Configuration discovery and precedence

Prettier walks **up** the directory tree from each file being formatted, and stops at the first config it finds (or at a `package.json` containing a `"prettier"` key, or at the repo root / home dir).

```
repo/
├── .prettierrc                    ← applies to everything below…
├── package.json                   ← …unless it has a "prettier" key (see below)
├── .prettierignore                ← ONLY the one at the CWD is read (see the trap)
├── packages/
│   ├── api/
│   │   ├── .prettierrc            ← …except here, which wins for packages/api/**
│   │   └── src/server.ts          ← formatted with packages/api/.prettierrc
│   └── web/
│       └── src/App.tsx            ← formatted with repo/.prettierrc
```

Accepted config file names, in the order Prettier checks them:

```
package.json  →  "prettier": { ... }  key
.prettierrc                            (JSON or YAML — Prettier sniffs)
.prettierrc.json
.prettierrc.yml / .prettierrc.yaml
.prettierrc.json5
.prettierrc.js / .prettierrc.cjs / .prettierrc.mjs
.prettierrc.ts / .prettierrc.mts / .prettierrc.cts   (Prettier 3.5+)
prettier.config.js / .cjs / .mjs / .ts
.prettierrc.toml
```

```ts
// ── prettier.config.ts — a typed config (Prettier 3.5+) ──────────────────────
// Why use a .ts config: autocomplete, and a COMPILE ERROR if you misspell an
// option instead of a silent no-op.
import type { Config } from "prettier";

// Sharing across repos: there is no "extends" key. Publish an npm package that
// exports a config object, then import and spread it — it's plain JS/TS.
import base from "@acme/prettier-config";

const config: Config = {
  ...base,
  printWidth: 100,          // this repo's one deviation
  overrides: [{ files: "*.md", options: { printWidth: 80, proseWrap: "preserve" } }],
};

export default config;
```

Two traps that cost people real time:

```
TRAP 1: .prettierignore is resolved relative to the CURRENT WORKING DIRECTORY,
        not relative to each file. Running `npx prettier --write .` from
        packages/api/ will NOT read repo-root/.prettierignore.
        Fix: run from the repo root, or pass --ignore-path explicitly:
          npx prettier --check . --ignore-path ../../.prettierignore

TRAP 2: Config does NOT merge across levels. A nested .prettierrc REPLACES the
        parent entirely — it does not inherit and override. If packages/api
        needs only printWidth changed, either import the root config in a .js
        config file, or use an `overrides` block in the root config instead.
```

### Concept 3 — `printWidth` is a target, not a rule

This is the option people misconfigure most, because they assume it behaves like a linter's `max-len`.

```ts
// printWidth: 80 (the default)

// Prettier's algorithm, simplified:
//   1. Try to print the whole construct on ONE line.
//   2. Does it fit in printWidth columns? → done.
//   3. It doesn't fit → break at the OUTERMOST group, then recurse into
//      the children, applying the same test to each.
//
// This means printWidth affects the SHAPE of your code, not just its width.

// ── At printWidth 80: doesn't fit → break every parameter ────────────────────
export async function reconcilePayments(
  tenantId: string,
  from: Date,
  to: Date,
): Promise<ReconciliationReport> {}

// ── At printWidth 120: fits → stays on one line ──────────────────────────────
export async function reconcilePayments(tenantId: string, from: Date, to: Date): Promise<ReconciliationReport> {}

// ── Things printWidth CANNOT break, and so will exceed it ────────────────────
const dsn = "postgres://user:password@some-very-long-hostname.rds.amazonaws.com:5432/production_db";
//     ↑ a string literal is atomic. Prettier will never split it.

type Handler = (req: Request, res: Response, next: NextFunction) => Promise<void> | void;
//     ↑ if this doesn't fit, Prettier breaks the PARAMETERS, but the type name
//       and the arrow stay put.

import { thisIsAVeryLongExportedIdentifierName } from "@acme/shared-domain-models";
//     ↑ single-specifier imports stay on one line even when over width.
```

Picking a value:

| printWidth | Fits | Trade-off |
|---|---|---|
| `80` | Prettier's default; side-by-side diffs on a 13" laptop | Verbose TS signatures wrap aggressively; lots of vertical space |
| `100` | The most common team choice for TS backends | Good balance; still fits two panes on a 1440p monitor |
| `120` | Wide monitors, dense NestJS/decorator code | Long lines are genuinely harder to scan; GitHub diff view wraps |
| `> 120` | Almost never | You lose the readability benefit that motivated the option |

For TypeScript backends, **100 is the pragmatic default**: TS signatures with generics and return types routinely land in the 85–100 column range, and 80 forces them to wrap in a way that adds noise without adding clarity.

### Concept 4 — The options that matter for TypeScript, one by one

```ts
// ═══ semi ════════════════════════════════════════════════════════════════════
// semi: true  (default)
const orderTotal = computeTotal(items);
export type OrderId = string;

// semi: false
const orderTotal = computeTotal(items)
export type OrderId = string
// Prettier still inserts a LEADING semicolon where ASI would break:
;[1, 2, 3].forEach(handle)
;(async () => { await main() })()
// ↑ This is the "defensive semicolon". It's why semi:false teams have code that
//   looks odd in exactly the places ASI is dangerous.

// ═══ trailingComma ═══════════════════════════════════════════════════════════
// "all"  (Prettier 3 default) — the one to use. See the deep dive below.
export async function createOrder(
  tenantId: string,
  items: LineItem[],
  options: CreateOrderOptions,   // ← trailing comma after the LAST param
): Promise<Order> {}

const config = {
  host: "localhost",
  port: 5432,                     // ← and in objects
};

type Status = 
  | "pending"
  | "paid";                       // ← unions do NOT get a trailing comma (no such syntax)

// "es5" — objects/arrays yes, function params no. The old default.
// "none" — nowhere. Produces the noisiest git diffs. Avoid.

// ═══ arrowParens ═════════════════════════════════════════════════════════════
// "always" (Prettier 2+ default)
const toId = (order: Order) => order.id;
const double = (n) => n * 2;

// "avoid"
const double = n => n * 2;
// ⚠️ In TypeScript this is mostly moot: the moment you add a type annotation,
//    parens are REQUIRED by the grammar. "avoid" only affects untyped params,
//    so in a strict TS codebase it applies to almost nothing. Use "always".

// ═══ quoteProps ══════════════════════════════════════════════════════════════
// "as-needed" (default) — quote only keys that must be quoted:
const a = { "content-type": "application/json", accept: "application/json" };
// "consistent" — if ONE key needs quotes, quote them all. Worth considering for
//                HTTP header maps and env-var objects, where mixed quoting reads badly:
const b = { "content-type": "application/json", "accept": "application/json" };
// "preserve" — leave exactly what you wrote (rare).

// ═══ endOfLine ═══════════════════════════════════════════════════════════════
// "lf" (default since Prettier 2) — normalises CRLF → LF on write. This single
// option prevents "the Windows dev's PR changed every line". Pair it with a
// .gitattributes containing:   * text=auto eol=lf

// ═══ bracketSpacing / useTabs / tabWidth ═════════════════════════════════════
// bracketSpacing true (default):  import { createOrder } from "./orderService.js";
//                false:           import {createOrder} from "./orderService.js";
// useTabs: false + tabWidth: 2 is the JS/TS norm. The real argument FOR tabs is
// accessibility: a low-vision user can set their own rendering width.

// ═══ embeddedLanguageFormatting ══════════════════════════════════════════════
// "auto" (default) — formats CSS/GraphQL/HTML inside RECOGNISED tagged template
// literals (gql``, css``, html``). An untagged `...` (e.g. raw SQL) is untouched.
// Set "off" if a tag name of yours collides with a language Prettier knows.

// ═══ objectWrap (Prettier 3.5+) ══════════════════════════════════════════════
// "preserve" (default) — if you put a newline after {, Prettier keeps the object
//   multi-line even when it would fit. This is the ONE place besides blank lines
//   where your input formatting is respected.
// "collapse" — always collapse to one line when it fits.
const point = {
  x: 1, y: 2,
};
// ↑ stays multi-line under "preserve", collapses to `const point = { x: 1, y: 2 };`
//   under "collapse".
```

### Concept 5 — Ignoring code Prettier gets wrong

Sometimes hand-alignment carries meaning — a matrix, an ASCII table, an aligned constant block. Three levels of escape hatch:

```ts
// ── Level 1: a single next node ──────────────────────────────────────────────
// prettier-ignore
const affineTransform = [
  1, 0, 0,
  0, 1, 0,
  0, 0, 1,
];
// Without the comment, Prettier collapses this to one line and destroys the shape.

// It works on any node, including type declarations:
// prettier-ignore
type HttpStatus =
  | 200 | 201 | 204
  | 400 | 401 | 403 | 404
  | 500 | 502 | 503;

// And on object literals used as lookup tables:
// prettier-ignore
const ERROR_CODES = {
  NOT_FOUND:     { status: 404, message: "Resource not found"  },
  UNAUTHORIZED:  { status: 401, message: "Missing credentials" },
  RATE_LIMITED:  { status: 429, message: "Too many requests"   },
};

// ⚠️ In JSX the syntax differs — it must be a JSX comment:
//   {/* prettier-ignore */}

// ── Level 2: a whole file, via .prettierignore ───────────────────────────────
// src/generated/**  — anything a code generator owns

// ── Level 3: a whole file, from inside the file ──────────────────────────────
// There is no file-level ignore comment in Prettier. If you need one, the file
// belongs in .prettierignore. (Some people wrap the entire file in a single
// // prettier-ignore before the first statement — it only covers that ONE node.)
```

Use `prettier-ignore` sparingly. Every instance is a small permanent exception a future reader must evaluate. If you find yourself adding many, the real question is usually whether that data belongs in a `.json` file instead of a `.ts` literal.

### Concept 6 — `eslint-config-prettier`: turning off the fights

Historically ESLint shipped stylistic rules: `indent`, `quotes`, `semi`, `comma-dangle`, `max-len`, `space-before-function-paren`, and `@typescript-eslint` mirrored many of them for TS syntax (`@typescript-eslint/indent`, `@typescript-eslint/member-delimiter-style`, and so on).

Run both without coordination and you get a loop:

```
1. You save.        → Prettier formats:      foo(  a,  b )  →  foo(a, b)
2. ESLint reports:  → "Expected indentation of 4 spaces but found 2"  (indent rule)
3. eslint --fix     → reindents to 4
4. You save again.  → Prettier reindents to 2
5. goto 2.
```

`eslint-config-prettier` is a config that contains **nothing but `"off"` switches**. It disables every ESLint rule that could possibly conflict with Prettier's output. It adds zero rules.

```bash
npm i -D eslint-config-prettier
```

```js
// ── eslint.config.js — FLAT CONFIG (ESLint 9+) ───────────────────────────────
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import prettierConfig from "eslint-config-prettier";

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    rules: {
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/consistent-type-imports": "error",
    },
  },
  prettierConfig,   // ← MUST BE LAST. It's a big list of "off"s; anything after
                    //    it could re-enable a conflicting rule.
);
```

```js
// ── .eslintrc.cjs — LEGACY ESLINTRC (ESLint 8 and below) ─────────────────────
module.exports = {
  parser: "@typescript-eslint/parser",
  parserOptions: { project: "./tsconfig.json", tsconfigRootDir: __dirname },
  plugins: ["@typescript-eslint"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended-type-checked",
    "prettier",          // ← eslint-config-prettier. LAST in the array.
  ],
  rules: {
    "@typescript-eslint/no-floating-promises": "error",
  },
};
```

To verify there's nothing left conflicting:

```bash
npx eslint-config-prettier src/server.ts
# Prints any enabled rules that conflict with Prettier. Clean output = you're done.
# (In older versions this was: npx eslint-config-prettier-check)
```

A relevant modernisation: **ESLint deprecated its core formatting rules in v8.53** and moved them to `@stylistic/eslint-plugin`. If your project is on a recent ESLint with no stylistic plugin installed, there is much less to disable — but `eslint-config-prettier` is still worth including, because plugins (React, Vue, import) still ship stylistic rules of their own.

### Concept 7 — Why running Prettier *as* an ESLint rule is discouraged

There is a package, `eslint-plugin-prettier`, that runs Prettier inside ESLint and reports every formatting difference as a lint error. It works. The Prettier team, and the ESLint team, both recommend against it as a default. Here is why, concretely.

```js
// ── The setup people copy from a blog post ───────────────────────────────────
module.exports = {
  extends: ["plugin:prettier/recommended"],
  // which expands to:
  //   plugins: ["prettier"],
  //   rules: { "prettier/prettier": "error" },
  //   extends: ["eslint-config-prettier"]
};
```

```
1 — SPEED. Every file is now parsed TWICE: once into an ESTree AST for ESLint's
    own rules, and again by Prettier's parser, which then prints and diffs the
    result character by character. On a 1,500-file TS service this routinely
    doubles or triples lint time — already the slowest CI step after tsc.

2 — NOISE. Every formatting difference is a red squiggle and a line of output.
    Open an unformatted file and you get 60 "errors", 58 of which say "insert a
    space". The floating promise and the unhandled union case get buried.

3 — CATEGORY CONFUSION. "Missing semicolon" now carries the same severity, in
    the same report, as "await has no effect on this type". One is a keystroke;
    one is a bug. Merging them trains people to skim lint output.

4 — WORSE MESSAGES. Prettier's CLI says "not formatted; run --write". The plugin
    says "Replace `·······` with `····`" — a diff of invisible characters.

5 — FRAGILE AUTOFIX ORDER. `eslint --fix` applies fixes in passes; a Prettier
    fix and a rule fix (say arrow-body-style) can undo each other, and in
    pathological cases ESLint bails with "fix loop detected".
```

```json
// ✅ The recommended arrangement — two independent tools, two independent gates
{ "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint . --max-warnings=0" } }
```

The legitimate exception: **an environment where you can only run one tool.** A locked-down editor integration, a CI system that only understands ESLint output, or a repo where you cannot add a second pre-commit step. In that case `eslint-plugin-prettier` is a reasonable trade — you're buying single-tool simplicity with slower runs and noisier output, knowingly.

### Concept 8 — Import organisation is not Prettier's job

Prettier will format an import statement — break a long specifier list, normalise quotes — but it will **never reorder imports**, and this is a deliberate, long-standing decision by the maintainers. Reordering imports can change behaviour (module side effects run in import order), so a formatter that promises to be behaviour-preserving cannot do it.

```ts
// Prettier's exact effect on imports: it will NOT touch this order.
import { createOrder } from "./services/orderService.js";
import express from "express";
import "./bootstrap/telemetry.js";   // ← a side-effect import. Moving this could
import { z } from "zod";             //   genuinely change program behaviour.
```

Your real options:

```js
// ── Option A: eslint-plugin-import (or eslint-plugin-import-x) ───────────────
// Pros: real ESLint rule, autofixable, configurable groups, no Prettier coupling.
// This is the recommended choice for most backend TS projects.
{
  plugins: { import: importPlugin },
  rules: {
    "import/order": ["error", {
      groups: [
        "builtin",     // node:fs, node:path
        "external",    // express, zod
        "internal",    // @/services/* — see pathGroups
        "parent",      // ../
        "sibling",     // ./
        "index",       // ./
        "type",        // import type { ... }
      ],
      pathGroups: [
        { pattern: "@acme/**", group: "internal", position: "before" },
        { pattern: "@/**",     group: "internal" },
      ],
      pathGroupsExcludedImportTypes: ["builtin"],
      "newlines-between": "always",
      alphabetize: { order: "asc", caseInsensitive: true },
      warnOnUnassignedImports: false,   // leaves side-effect imports where they are
    }],
  },
}
```

```jsonc
// ── Options B/C: a Prettier sort PLUGIN — runs in the format step, on save ──
// @trivago/prettier-plugin-sort-imports  — the original; can reorder
//   side-effect imports unless configured carefully.
// @ianvs/prettier-plugin-sort-imports    — maintained fork, better TS support
//   (handles `import type`, leaves side-effect imports alone by default).
{
  "plugins": ["@ianvs/prettier-plugin-sort-imports"],
  "importOrder": [
    "^node:", "<BUILTIN_MODULES>",
    "",                        // "" = insert a blank line separator here
    "<THIRD_PARTY_MODULES>",
    "",
    "^@acme/(.*)$",
    "",
    "^[./]"
  ],
  "importOrderTypeScriptVersion": "5.6.0"
}
```

```jsonc
// ── Option D: VS Code / TS language service "Organize Imports" ──────────────
// Editor-only. Sorts AND removes unused imports. But it is NOT enforceable in
// CI (there's no CLI), so it drifts. Good as a convenience, bad as a policy.
{
  "editor.codeActionsOnSave": {
    "source.organizeImports": "explicit"
  }
}
// ⚠️ If you combine this with a Prettier sort plugin, they can disagree and
//    fight on every save. Pick ONE import authority. This is the single most
//    common cause of "my editor keeps changing my imports back and forth".
```

Recommendation for a backend TS service: **Option A**. Import order is a code convention with correctness implications, which puts it squarely in ESLint's domain, and it's the one option where the fix runs in the same pass as your other autofixes.

### Concept 9 — Formatting is not a lint error, and CI should say so separately

This is a conceptual point with a practical payoff.

```
A LINT ERROR says:   "This code will probably misbehave."
                     → requires a human to think, understand, and decide.
                     → e.g. no-floating-promises, no-unsafe-assignment.

A FORMAT DIFF says:  "This code is spelled differently than the tool spells it."
                     → requires nobody to think. A machine fixes it in 0.2s.
                     → e.g. two spaces vs four.
```

Because they are different kinds of failure, they deserve different treatment:

- **Different CI steps**, so the log tells you which failed at a glance.
- **Different fix instructions**: `npm run format` vs "read this rule's docs".
- **Different severities in review**: a format diff is never a review comment; a lint error might be.
- **Different blocking policy**: some teams auto-fix formatting on the CI branch and push; nobody auto-fixes lint errors.

```yaml
# ── The shape of the CI job: four separately-named steps, cheapest first ─────
# (full workflow in Example 2)
      - name: Check formatting     # ~2–5s. Fails fast on whitespace-only issues.
        run: npx prettier --check .
      - name: Lint                 # correctness only — can never fail for style
        run: npx eslint . --max-warnings=0
      - name: Typecheck            # the slow one, so it runs later
        run: npx tsc --noEmit
      - name: Test
        run: npm test
```

`prettier --check` output is designed for exactly this:

```
$ npx prettier --check .
Checking formatting...
[warn] src/services/orderService.ts
[warn] src/routes/webhookRoutes.ts
[warn] Code style issues found in 2 files. Run Prettier with --write to fix.
$ echo $?
1
```

There is also `--list-different` (alias `-l`), which prints only the filenames and nothing else — handy for piping:

```bash
npx prettier --list-different "src/**/*.ts" | xargs -r npx prettier --write
```

---

## Example 1 — basic

A complete, minimal Prettier setup on a fresh TypeScript Node project.

```bash
# ── Step 1: install, pinned ──────────────────────────────────────────────────
mkdir prettier-lab && cd prettier-lab
npm init -y
npm i -D --save-exact prettier
npm i -D typescript @types/node
npx tsc --init
```

```jsonc
// ── Step 2: .prettierrc ─────────────────────────────────────────────────────
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

```
# ── Step 3: .prettierignore ─────────────────────────────────────────────────
dist/
coverage/
node_modules/
package-lock.json
```

```ts
// ── Step 4: src/orderService.ts — deliberately badly formatted ──────────────
// Type this in with the mess intact, so you can watch Prettier fix it.

export interface LineItem { sku : string ; unitPriceCents : number ; quantity : number }

export interface Order { orderId : string ,
        tenantId : string ,
   items : LineItem [] ,
      status : "pending" | "paid" | "shipped" | "cancelled" }

export function computeOrderTotalCents( order : Order ) : number {
    return order.items.reduce( ( sum , item ) => sum + item.unitPriceCents * item.quantity , 0 )
}

export function summarise(order:Order):string{
  const total=computeOrderTotalCents(order);return `Order ${order.orderId} (${order.status}): ${order.items.length} items, ${(total/100).toFixed(2)}`
}
```

```bash
# ── Step 5: see what WOULD change, without changing it ──────────────────────
npx prettier --check src/orderService.ts
# [warn] src/orderService.ts
# [warn] Code style issues found in 1 file. Run Prettier with --write to fix.

# See the actual output without writing (useful for understanding a decision):
npx prettier src/orderService.ts        # prints formatted result to stdout

# ── Step 6: format it ───────────────────────────────────────────────────────
npx prettier --write src/orderService.ts
# src/orderService.ts 34ms
```

```ts
// ── Step 7: the result ──────────────────────────────────────────────────────
export interface LineItem {
  sku: string;
  unitPriceCents: number;
  quantity: number;
}
// ↑ Note: Prettier chose SEMICOLONS as the interface member delimiter, not
//   commas. It normalises `,` → `;` inside interfaces and object type literals.
//   This is not configurable in Prettier (it was `member-delimiter-style` in
//   the old ESLint stylistic rules).

export interface Order {
  orderId: string;
  tenantId: string;
  items: LineItem[];
  status: "pending" | "paid" | "shipped" | "cancelled";
}
// ↑ The union fits within printWidth 100, so it stays on one line. At
//   printWidth 40 it would become:
//     status:
//       | "pending"
//       | "paid"
//       | "shipped"
//       | "cancelled";
//   Leading-pipe style — that's Prettier's format for a broken union.

export function computeOrderTotalCents(order: Order): number {
  return order.items.reduce((sum, item) => sum + item.unitPriceCents * item.quantity, 0);
}
// ↑ 92 characters — fits in 100, so no wrapping. At printWidth 80:
//     return order.items.reduce(
//       (sum, item) => sum + item.unitPriceCents * item.quantity,
//       0,
//     );

export function summarise(order: Order): string {
  const total = computeOrderTotalCents(order);
  return `Order ${order.orderId} (${order.status}): ${order.items.length} items, ${(total / 100).toFixed(2)}`;
}
// ↑ TWO things happened here:
//   1. The `const...;return...` one-liner was split into two statements.
//   2. The template literal is 108 chars — OVER printWidth — and Prettier left
//      it alone, because it cannot break a template literal without changing
//      the string's value. printWidth is a target, not a guarantee.
```

```jsonc
// ── Step 8: .vscode/settings.json — format on save ──────────────────────────
{ "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "prettier.requireConfig": true }

// ── Step 9: package.json scripts, so it isn't editor-only ───────────────────
{ "scripts": { "format": "prettier --write .", "format:check": "prettier --check ." } }
```

Now do the thing that makes it real: run `npm run format` once across the whole repo, commit that in a **single commit with no other changes**, and add that commit's SHA to `.git-blame-ignore-revs` (covered in "Going deeper").

---

## Example 2 — real world backend use case

A production-shaped Express + Postgres service with Prettier, ESLint, husky, lint-staged, and CI — the arrangement that actually holds up on a team.

```
acme-orders-api/
├── .github/workflows/ci.yml
├── .husky/pre-commit
├── .vscode/
│   ├── settings.json
│   └── extensions.json
├── .git-blame-ignore-revs
├── .prettierrc
├── .prettierignore
├── .editorconfig
├── .gitattributes
├── eslint.config.js
├── package.json
├── tsconfig.json
└── src/
    ├── server.ts
    ├── config.ts
    ├── routes/orderRoutes.ts
    ├── services/orderService.ts
    ├── db/pool.ts
    └── generated/prisma-client.ts   ← ignored by Prettier
```

```jsonc
// ── .prettierrc ──────────────────────────────────────────────────────────────
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "always",
  "bracketSpacing": true,
  "quoteProps": "as-needed",
  "endOfLine": "lf",
  "embeddedLanguageFormatting": "auto",

  "overrides": [
    {
      // Markdown: narrower for readability, and never reflow prose — reflowing
      // turns a one-word edit into a 12-line diff.
      "files": ["*.md", "*.mdx"],
      "options": { "printWidth": 80, "proseWrap": "preserve" }
    },
    {
      // YAML: 2-space indent is mandatory in most CI schemas; also, YAML is
      // whitespace-significant, so keep printWidth generous to avoid rewraps.
      "files": ["*.yml", "*.yaml"],
      "options": { "tabWidth": 2, "printWidth": 120, "singleQuote": false }
    },
    {
      // JSON files have no trailing-comma syntax. Prettier knows this for .json,
      // but be explicit for .prettierrc / .babelrc style extensionless files.
      "files": [".prettierrc", ".babelrc", "*.json"],
      "options": { "parser": "json", "trailingComma": "none" }
    },
    {
      // Test files: wider, because `expect(...).toEqual({ ...big object })`
      // wraps into unreadable confetti at 100.
      "files": ["*.test.ts", "*.spec.ts", "**/__tests__/**/*.ts"],
      "options": { "printWidth": 120 }
    }
  ]
}
```

```
# ── .prettierignore ─────────────────────────────────────────────────────────
dist/                        # build artefacts
build/
coverage/
.turbo/
node_modules/

# Code generators own their formatting. Reformatting means every regeneration
# produces a diff, which defeats the point of generating.
src/generated/
prisma/migrations/
src/graphql/__generated__/
*.gen.ts
*.pb.ts
openapi.generated.json

package-lock.json            # lockfiles
pnpm-lock.yaml
**/__snapshots__/            # reformatting invalidates every snapshot at once
src/__fixtures__/raw/        # fixtures that must stay byte-exact
```

```js
// ── eslint.config.js — flat config, Prettier last ───────────────────────────
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import importPlugin from "eslint-plugin-import";
import prettierConfig from "eslint-config-prettier";

export default tseslint.config(
  { ignores: ["dist/**", "coverage/**", "src/generated/**"] },

  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,

  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    plugins: { import: importPlugin },
    rules: {
      // ── CORRECTNESS: things Prettier can never see ──────────────────────
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      "@typescript-eslint/await-thenable": "error",
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/switch-exhaustiveness-check": "error",
      "@typescript-eslint/consistent-type-imports": [
        "error",
        { prefer: "type-imports", fixStyle: "inline-type-imports" },
      ],

      // ── CONVENTION: import ORDER — Prettier explicitly won't do this ────
      "import/order": [
        "error",
        {
          groups: ["builtin", "external", "internal", "parent", "sibling", "index", "type"],
          pathGroups: [{ pattern: "@/**", group: "internal" }],
          "newlines-between": "always",
          alphabetize: { order: "asc", caseInsensitive: true },
        },
      ],
      "import/no-duplicates": "error",
    },
  },

  // ← LAST. Disables every ESLint rule that could disagree with Prettier.
  //   Note: this does NOT disable import/order, because import/order does not
  //   conflict with Prettier — Prettier has no opinion on import ordering.
  prettierConfig,
);
```

```json
// ── package.json ────────────────────────────────────────────────────────────
{
  "name": "acme-orders-api",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc -p tsconfig.json",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint . --max-warnings=0",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "verify": "npm run format:check && npm run lint && npm run typecheck && npm test",
    "prepare": "husky"
  },
  "devDependencies": {
    "prettier": "3.4.2",              // ← EXACT, no caret
    "eslint": "^9.17.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-import": "^2.31.0",
    "typescript-eslint": "^8.18.0",
    "husky": "^9.1.7",
    "lint-staged": "^15.2.11",
    "typescript": "^5.7.2"
  },
  "lint-staged": {
    // KEY = glob of STAGED files. VALUE = command(s) run with those paths appended.
    // lint-staged re-stages whatever the commands modify, so --write is safe here.
    "*.{ts,tsx,mts,cts,js,mjs,cjs}": ["prettier --write", "eslint --fix --max-warnings=0"],
    "*.{json,jsonc,md,mdx,yml,yaml,css,html}": ["prettier --write"]
  }
}
```

```bash
# ── husky (v9) ──────────────────────────────────────────────────────────────
npm i -D husky lint-staged
npx husky init      # creates .husky/ and adds "prepare": "husky" to package.json

# .husky/pre-commit   (v9 needs no shebang and no husky.sh sourcing line)
npx lint-staged

# .husky/pre-push — the heavy checks. pre-commit runs on EVERY commit and must
# stay under ~3s or people start using --no-verify; tsc takes 15–40s.
npm run typecheck
```

```yaml
# ── .github/workflows/ci.yml ────────────────────────────────────────────────
name: CI
on:
  push: { branches: [main] }
  pull_request:
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: "npm" }
      - run: npm ci

      # ── Formatting: 2–5 seconds. Runs first so a whitespace-only failure
      #    doesn't wait behind a 40-second tsc run.
      #    NOTE: this must run from the REPO ROOT so .prettierignore is found.
      - name: Check formatting
        run: npx prettier --check .

      # ── Lint: correctness only. eslint-config-prettier guarantees this can
      #    never fail for a formatting reason.
      - name: Lint
        run: npx eslint . --max-warnings=0

      - name: Typecheck
        run: npx tsc --noEmit

      - name: Test
        run: npm test -- --coverage
```

```jsonc
// ── .vscode/settings.json — committed, so the team is consistent ────────────
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,

  // Per-language pins. Necessary because other extensions register themselves
  // as formatters for these languages and can silently win the default.
  "[typescript]":      { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[typescriptreact]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[javascript]":      { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[json]":            { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[yaml]":            { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[markdown]":        { "editor.defaultFormatter": "esbenp.prettier-vscode" },

  // Only format repos that have a Prettier config. Without this, opening an
  // unrelated repo and hitting save reformats a file with YOUR defaults.
  "prettier.requireConfig": true,

  // Use the repo's pinned prettier from node_modules, not the version bundled
  // in the extension. Prevents "works on my machine, fails in CI".
  "prettier.prettierPath": "./node_modules/prettier",

  // Run eslint --fix on save too. VS Code runs formatOnSave first, then code
  // actions — so Prettier formats, then ESLint fixes correctness issues.
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },

  // Do NOT also enable source.organizeImports here: it fights import/order.
  // Pick one import authority. Ours is eslint-plugin-import.

  "files.eol": "\n",
  "files.insertFinalNewline": true,
  "files.trimTrailingWhitespace": true
}
```

```ini
# ── .editorconfig — Prettier READS this; .prettierrc values win over it ─────
root = true
[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 2
max_line_length = 100
[*.md]
trim_trailing_whitespace = false   # two trailing spaces = <br> in Markdown

# ── .gitattributes — stop CRLF from ever entering the repo ──────────────────
# * text=auto eol=lf
```

```ts
// ── src/services/orderService.ts — after formatting ─────────────────────────
import { randomUUID } from "node:crypto";

import { z } from "zod";

import { pool } from "@/db/pool.js";
import type { LineItem, Order, OrderStatus } from "@/domain/types.js";

const createOrderSchema = z.object({
  tenantId: z.string().uuid(),
  customerId: z.string().uuid(),
  items: z
    .array(
      z.object({
        sku: z.string().min(1),
        unitPriceCents: z.number().int().positive(),
        quantity: z.number().int().positive(),
      }),
    )
    .min(1),
  idempotencyKey: z.string().optional(),
});
// ↑ Look at what Prettier did with the chained zod builder: because the whole
//   chain doesn't fit, it broke at the MEMBER ACCESS points and indented.
//   Note `z` alone on the first line — that's Prettier's "the first call in the
//   chain is short, so hang it" heuristic. You cannot configure this. It is
//   the single most common thing people file issues about, and the answer is
//   always the same: Prettier's line-breaking is not configurable by design.

export type CreateOrderInput = z.infer<typeof createOrderSchema>;

export async function createOrder(input: CreateOrderInput): Promise<Order> {
  const parsed = createOrderSchema.parse(input);
  const orderId = randomUUID();

  const totalCents = parsed.items.reduce(
    (sum, item) => sum + item.unitPriceCents * item.quantity,
    0,
  );
  // ↑ trailingComma: "all" put a comma after `0`. That's the config choice
  //   that makes adding a fourth argument a ONE-line diff instead of two.

  const { rows } = await pool.query<Order>(
    `INSERT INTO orders (order_id, tenant_id, customer_id, total_cents, status)
     VALUES ($1, $2, $3, $4, $5)
     RETURNING *`,
    [orderId, parsed.tenantId, parsed.customerId, totalCents, "pending" satisfies OrderStatus],
  );
  // ↑ The SQL inside the template literal is NOT formatted, because the literal
  //   is untagged. Tag it with a `sql` tag and install a Prettier SQL plugin if
  //   you want that — most teams don't bother.

  const order = rows[0];
  if (!order) {
    throw new Error(`INSERT returned no row for order ${orderId}`);
  }

  return order;
}
```

Day-to-day, the loop is: you write code however you like, hit save, VS Code formats it. If you forget, `git commit` triggers lint-staged which formats the staged files and re-stages them. If both somehow fail, CI catches it in five seconds with an exact fix command. Formatting never reaches a human reviewer.

---

## Going deeper

### The `.git-blame-ignore-revs` file — surviving the initial reformat

Adopting Prettier on an existing codebase means one commit that touches every file. That commit poisons `git blame` — every line now blames "chore: apply prettier".

```bash
# 1. Do the reformat as its OWN commit, containing nothing else:
npx prettier --write .
git add -A
git commit -m "chore: format entire codebase with prettier"
git rev-parse HEAD
# → 9f3a71c4e8b2d5a1f0c6e9b3d7a2c5f8e1b4d6a9
```

```
# 2. .git-blame-ignore-revs  (repo root, committed)
# Bulk reformat commits — ignored by `git blame --ignore-revs-file`.
# Format: one full 40-char SHA per line, comments with #.

# chore: format entire codebase with prettier (2026-01-14)
9f3a71c4e8b2d5a1f0c6e9b3d7a2c5f8e1b4d6a9

# chore: bump prettier 3.3 → 3.4, reformat (2026-04-02)
2c7e5f1a9d4b8c3e6f0a5d2b7c9e4f1a8d6b3c50
```

```bash
# 3. Make it automatic for everyone on the team:
git config blame.ignoreRevsFile .git-blame-ignore-revs
# Add that command to your README. GitHub's blame view reads the file
# automatically with no configuration.
```

### The semicolon argument, settled

The `semi` option is the single most-argued Prettier setting. Both positions are defensible; here is the actual technical content, so you can end the discussion in your team in five minutes rather than fifty.

```ts
// ═══ THE CASE FOR semi: true (Prettier's default) ═══════════════════════════

// 1. It is what the TypeScript team, the Prettier default, and the majority of
//    the ecosystem use. Copy-pasting from docs/StackOverflow just works.

// 2. ASI (Automatic Semicolon Insertion) has genuine footguns. With semi: false,
//    Prettier protects you by inserting LEADING semicolons — which is arguably
//    uglier than just writing trailing ones:
const items = getItems()
;[...items].sort()          // ← without the leading ;, this parses as getItems()[...]
;(async () => {})()         // ← without it, this parses as a call on the previous line

// 3. The classic ASI hazard is unaffected by the `semi` option either way,
//    because it happens at PARSE time, before Prettier prints anything:
function bad() {
  return                     // ASI ends the statement HERE
    { ok: true };            // a block, never reached — the function returns undefined
}
// Prettier reprints from the AST, so it emits `return;` on its own line and the
// block below it — making the bug VISIBLE. That's a point for Prettier in
// general, not for `semi: true` specifically.

// ═══ THE CASE FOR semi: false ═══════════════════════════════════════════════

// 1. Less visual noise; the character carries no information in 99.9% of lines.
// 2. Prettier handles the dangerous cases mechanically, so ASI risk ≈ 0.
// 3. Popular in the Vue / Nuxt / standard.js ecosystems — if your team lives
//    there, matching the surrounding ecosystem has real value.

// ═══ THE ANSWER ═════════════════════════════════════════════════════════════
// It does not matter. Pick one in your first week, write it in .prettierrc,
// and never discuss it again. If you have no strong opinion, take the default
// (semi: true) — the value of matching the ecosystem exceeds the value of
// either aesthetic.
```

### The trailing-comma argument, settled (this one has a right answer)

Unlike semicolons, `trailingComma` has a defensible objectively-better setting, and it's `"all"`.

```ts
// ═══ WHY "all" WINS: the git diff ═══════════════════════════════════════════

// With trailingComma: "none" — adding a parameter touches TWO lines:
export async function createOrder(
  tenantId: string,
  customerId: string    // ← line changed ONLY to add a comma
+ ,
+ idempotencyKey: string
): Promise<Order> {}
// Diff:  - customerId: string
//        + customerId: string,
//        + idempotencyKey: string

// With trailingComma: "all" — adding a parameter touches ONE line:
export async function createOrder(
  tenantId: string,
  customerId: string,
+ idempotencyKey: string,
): Promise<Order> {}
// Diff:  + idempotencyKey: string,

// This compounds: cleaner diffs → cleaner git blame → fewer merge conflicts
// (two people adding a parameter to the same function no longer conflict on the
// shared "customerId" line).

// ═══ THE HISTORICAL OBJECTION, AND WHY IT NO LONGER APPLIES ═════════════════

// "es5" was the Prettier 2 default because trailing commas in FUNCTION
// parameter lists and CALL arguments were an ES2017 feature. On old browsers or
// Node < 8, `foo(a, b,)` was a syntax error.
//
// Today: every runtime you target supports it, and TypeScript compiles it away
// for old targets anyway. Prettier 3 changed the default to "all" for this
// exact reason.

// ═══ WHERE TRAILING COMMAS ARE STILL ILLEGAL — Prettier knows ═══════════════
// JSON:  { "a": 1, }              ← syntax error. Prettier never adds one to .json
// Rest params: (a, ...rest,) => {} ← syntax error. Prettier never adds one here.
const fn = (first: string, ...rest: number[]) => rest.length;   // no trailing comma ✅

// Type parameter lists in .tsx files have a related quirk that predates this:
const identity = <T,>(value: T): T => value;
//                   ^ the comma disambiguates a generic arrow from a JSX tag.
//                   Prettier preserves it in .tsx and drops it in .ts.

// ═══ THE ANSWER ═════════════════════════════════════════════════════════════
// "all". Use "es5" only if you have a build target that genuinely predates
// ES2017 and no transpiler in the chain. Never use "none".
```

### `prettier --check` vs `--list-different` vs `--write` in CI

```bash
# --check    : prints [warn] lines + a summary, exits 1 if anything differs.
#              THE ONE TO USE IN CI. Human-readable, obvious fix instruction.
npx prettier --check .

# --list-different (-l) : prints ONLY filenames, exits 1 if any. For scripting.
npx prettier -l .

# --write (-w) : rewrites files in place, exits 0 (unless a parse error).
#              NEVER use this as your CI check — it "passes" by mutating the
#              checkout, so the check can literally never fail.
npx prettier --write .        # ❌ as a CI gate

# --cache : Prettier 3 caches results keyed by content+config in
#           node_modules/.cache/prettier/.prettier-cache. Big speedup on
#           re-runs. Combine with actions/cache for CI.
npx prettier --check . --cache --cache-strategy content

# --no-error-on-unmatched-pattern : don't fail when a glob matches nothing.
#           Essential in scripts that pass a dynamic file list.
npx prettier --check "src/**/*.tsx" --no-error-on-unmatched-pattern
```

An anti-pattern worth naming explicitly: **CI jobs that run `prettier --write` and push a "style fix" commit back to the PR branch.** It seems helpful. In practice it (a) invalidates the contributor's local checkout, causing confusing force-push conflicts, (b) makes the CI bot a co-author of every PR, and (c) hides the fact that a developer's editor isn't set up. Fail the build instead; the fix is one command.

### lint-staged, and why the pre-commit hook must be fast

The rule people learn the hard way: **if the pre-commit hook takes more than a couple of seconds, developers start typing `git commit --no-verify`, and your hook is now decorative.**

```json
// ✅ Fast pre-commit: formats and lints only the STAGED files.
{ "lint-staged": {
    "*.{ts,tsx}": ["prettier --write", "eslint --fix --max-warnings=0"],
    "*.{json,md,yml,css}": ["prettier --write"] } }
```

```json
// ❌ Slow AND wrong: type-checking inside the staged-files glob.
{ "lint-staged": { "*.ts": ["prettier --write", "eslint --fix", "tsc --noEmit"] } }
// 1. Slow: 15–60s on a real service, on every commit.
// 2. WRONG: lint-staged APPENDS the staged filenames to each command, so this
//    runs `tsc --noEmit src/a.ts src/b.ts` — and passing files to tsc makes it
//    IGNORE tsconfig.json entirely. You get a typecheck with default compiler
//    options (no strict, no paths), which is not the check you wanted.
```

```js
// ✅ If you must typecheck in a hook, use the FUNCTION form, which discards
//    the file list — and put it on pre-push, not pre-commit.
// lint-staged.config.js
export default {
  "*.{ts,tsx}": ["prettier --write", "eslint --fix --max-warnings=0"],
  "*.ts": () => "tsc --noEmit -p tsconfig.json",   // ← ignores the file list
};
```

Four more things worth knowing:

```
• lint-staged STASHES unstaged changes before running and restores them after.
  If a command crashes hard it can leave the stash behind — `git stash list`
  is where to look when someone says "my changes vanished".
• It RE-STAGES whatever the commands modified, so the formatted version is what
  actually gets committed. That's why `--write` is safe here.
• Order within the array matters: `prettier --write` FIRST, then `eslint --fix`.
• `--max-warnings=0` on the eslint step turns warnings into a failed commit;
  without it a "warn" rule passes the hook and then fails CI.

Escape hatches:  git commit --no-verify     (fine for local WIP you'll squash)
                 HUSKY=0 git commit         (husky v9 env-var kill switch)
```

### Prettier plugins — the small ecosystem worth knowing

```jsonc
// npm i -D prettier-plugin-organize-imports   # uses the TS language service
//          @ianvs/prettier-plugin-sort-imports # community sorter, TS-aware
//          prettier-plugin-tailwindcss         # sorts Tailwind class strings
//          prettier-plugin-packagejson         # canonical package.json key order
//          prettier-plugin-sql                 # formats tagged sql`` templates
// .prettierrc — plugins load IN ORDER, and order matters:
{ "plugins": ["prettier-plugin-organize-imports", "prettier-plugin-tailwindcss"] }
//                                                 ↑ tailwind must be LAST
```

`prettier-plugin-organize-imports` is a notable one for TS specifically: it delegates to the **TypeScript language service's own** organize-imports implementation — the same code behind VS Code's "Organize Imports" command. That means it also **removes unused imports**, which is a code change, not a format change. Some teams love that; others consider it out of scope for a formatter. Know which you're signing up for.

```
⚠️ prettier-plugin-organize-imports is INCOMPATIBLE with the sort-imports
   plugins — they both claim the import section and produce nondeterministic
   results depending on load order. Pick exactly one import authority across
   your whole toolchain: Prettier plugin, OR eslint-plugin-import, OR the
   editor's organizeImports action. Never two.
```

### Performance, and `.editorconfig` precedence

```bash
# Prettier is single-threaded and does ~1,000–3,000 TS files/sec. A cold
# `--check .` on a 20k-file monorepo is ~10–20s. Three fixes, in order:

npx prettier --check . --cache --cache-strategy content
# Content-hash cache at node_modules/.cache/prettier/.prettier-cache.
# Persist it in CI with actions/cache keyed on .prettierrc + the lockfile.

# 2. Scope to changed files (git diff --name-only | xargs npx prettier --check).
# 3. Make .prettierignore aggressive — generated code, fixtures, snapshots.
# If it still blocks you, Biome (Rust, ~95% Prettier-compatible) and dprint are
# the faster alternatives; both need a one-time reformat commit to adopt.
```

Prettier also reads `.editorconfig`, mapping `indent_style` → `useTabs`, `indent_size`/`tab_width` → `tabWidth`, `max_line_length` → `printWidth`, `end_of_line` → `endOfLine`. **`.prettierrc` wins; `.editorconfig` only fills gaps.** The failure mode: `.editorconfig` says `indent_size = 4`, `.prettierrc` is silent on `tabWidth`, and someone spends an hour hunting in `.prettierrc` for a setting that isn't there. Be explicit in `.prettierrc` about everything that matters.

### What Prettier will never let you configure, and why

Prettier's option list is deliberately tiny and closed. The maintainers publish an explicit rationale: every option is a decision the team has to make and re-litigate, and the value of a formatter comes from removing decisions, not offering them.

Things people ask for and will not get:

```
❌ Vertical alignment of object values / assignments / trailing comments
❌ Configurable line-breaking heuristics for method chains
❌ "Keep this call on one line even though it's over printWidth"
❌ Blank line rules (e.g. "always one blank line after imports")
❌ Sorting of anything: imports, object keys, union members, class members
❌ Brace style (Allman/K&R) — K&R only
❌ Configurable indentation of ternaries, JSX, or chained calls
❌ Maximum blank lines between statements (hardcoded to 1)
```

If any of these are non-negotiable for your team, Prettier is the wrong tool and `dprint` (highly configurable) or `@stylistic/eslint-plugin` (rule-per-decision) are the honest alternatives. Adopting Prettier is accepting its opinions. That acceptance is the product.

---

## Common mistakes

### Mistake 1 — Running Prettier as an ESLint rule by default

```js
// ❌ WRONG (as a default choice): Prettier runs inside ESLint on every file.
// .eslintrc.cjs
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:prettier/recommended",   // ← eslint-plugin-prettier
  ],
};
// Consequences:
//   • Lint time doubles or triples (double parse per file).
//   • "Delete `··`" appears as an ERROR alongside real bugs.
//   • Editor squiggles for whitespace bury the squiggles that matter.
//   • eslint --fix can enter multi-pass fix loops with other autofixable rules.
```

```js
// ✅ RIGHT: two tools, two jobs, run separately.
// eslint.config.js
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import prettierConfig from "eslint-config-prettier";   // ONLY the "off" switches

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  prettierConfig,   // ← last
);

// package.json — separate scripts, separate CI steps:
//   "format:check": "prettier --check ."
//   "lint":         "eslint . --max-warnings=0"
```

### Mistake 2 — Expecting Prettier to sort imports (or remove unused ones)

```ts
// ❌ WRONG assumption: "we have Prettier, so imports are organised."
// This file is 100% Prettier-clean. Prettier will never reorder it.
import { createOrder } from "./services/orderService.js";
import express from "express";
import { unusedThing } from "./utils.js";     // ← unused; Prettier keeps it
import { randomUUID } from "node:crypto";
import { z } from "zod";
```

```js
// ✅ RIGHT: pick ONE import authority and configure it explicitly.

// Option A (recommended for backends) — eslint-plugin-import:
{
  rules: {
    "import/order": ["error", {
      groups: ["builtin", "external", "internal", "parent", "sibling", "index", "type"],
      "newlines-between": "always",
      alphabetize: { order: "asc", caseInsensitive: true },
    }],
  },
}
// And for unused imports, that's a separate rule entirely:
{
  rules: { "@typescript-eslint/no-unused-vars": ["error", { args: "after-used" }] },
}
// (or `noUnusedLocals: true` in tsconfig.json — see 03)
```

```ts
// ✅ The result — and note that ESLint, not Prettier, produced this ordering:
import { randomUUID } from "node:crypto";

import express from "express";
import { z } from "zod";

import { createOrder } from "./services/orderService.js";
```

### Mistake 3 — Not pinning the Prettier version

```json
// ❌ WRONG: a caret range. Prettier ships intentional formatting changes in
//    minor releases (they document them as such). One developer runs 3.4.2,
//    another's fresh `npm install` pulls 3.6.0, and now every file they touch
//    reformats. CI fails on PRs that changed nothing relevant.
{ "devDependencies": { "prettier": "^3.4.2" } }
```

```json
// ✅ RIGHT: exact version, upgraded deliberately.
{ "devDependencies": { "prettier": "3.4.2" } }
// Install with:  npm i -D --save-exact prettier
//
// Upgrading is then a deliberate, reviewable event:
//   1. npm i -D --save-exact prettier@3.6.0
//   2. npx prettier --write .
//   3. Commit the reformat SEPARATELY from the version bump? No — commit them
//      TOGETHER, so the tree is never inconsistent with package.json.
//   4. Add the SHA to .git-blame-ignore-revs.
```

```jsonc
// ✅ And make sure the EDITOR uses the pinned copy, not the extension's bundle:
// .vscode/settings.json
{
  "prettier.prettierPath": "./node_modules/prettier"
}
```

### Mistake 4 — Running `prettier --write` in CI as the "check"

```yaml
# ❌ WRONG: this can never fail. Prettier rewrites the checkout and exits 0.
#    The job is green forever and enforces nothing.
- name: Format
  run: npx prettier --write .
```

```yaml
# ❌ ALSO WRONG (a more subtle version): format, then commit back to the PR.
- run: npx prettier --write .
- run: |
    git config user.name "ci-bot"
    git commit -am "style: prettier" && git push
# Problems: force-push conflicts for the contributor, CI bot as co-author on
# every PR, and it hides the fact that someone's editor isn't configured.
```

```yaml
# ✅ RIGHT: --check. Fails fast, tells you exactly which files and how to fix.
- name: Check formatting
  run: npx prettier --check .
# Output on failure:
#   [warn] src/services/orderService.ts
#   [warn] Code style issues found in 1 file. Run Prettier with --write to fix.
# Exit code 1 → build red → developer runs `npm run format` → done.
```

### Mistake 5 — Wrong or missing `defaultFormatter`, so format-on-save does nothing (or the wrong thing)

```jsonc
// ❌ WRONG: formatOnSave with no defaultFormatter. VS Code has several
//    extensions registered as TypeScript formatters (the built-in TS one,
//    Prettier, ESLint, Biome). It either picks one arbitrarily, or shows
//    "There are multiple formatters for TypeScript files" and does nothing.
{ "editor.formatOnSave": true }

// ❌ ALSO WRONG: a display name instead of the extension ID.
{ "editor.defaultFormatter": "Prettier" }   // ID is "esbenp.prettier-vscode"
```

```jsonc
// ✅ RIGHT: explicit ID, plus per-language pins so no other extension can win.
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "[typescript]":      { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[typescriptreact]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[json]":            { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "prettier.requireConfig": true,
  "prettier.prettierPath": "./node_modules/prettier"
}
```

```
Debugging format-on-save when it silently doesn't work:
  1. Command Palette → "Format Document With…" → is Prettier even listed?
  2. Output panel → dropdown → "Prettier" → it logs the config it resolved
     and any parse errors.
  3. `npx prettier --file-info src/thing.ts` → is the file ignored? which parser?
  4. Check the file isn't matched by .prettierignore.
  5. Check "prettier.requireConfig": true isn't firing because you have no config.
```

### Mistake 6 — Treating `printWidth` as `max-len`, then adding an ESLint rule to enforce it

```js
// ❌ WRONG: Prettier can't always honour printWidth (long strings, long
//    identifiers, single-specifier imports). max-len then produces errors that
//    are literally unfixable without hand-editing code Prettier just wrote.
{ rules: { "max-len": ["error", { code: 100 }] } }   // fights Prettier forever
// eslint-config-prettier turns max-len OFF precisely because of this — if you
// re-enable it afterwards, you have broken the setup you just installed.
```

```ts
// ✅ RIGHT: printWidth is the ONLY line-length authority; no max-len in ESLint.
//    If one long line genuinely hurts, fix it at the source:
const DEFAULT_DATABASE_URL =
  "postgres://user:password@some-very-long-hostname.rds.amazonaws.com:5432/production_db";
// ↑ Prettier broke after the `=`. That's its answer for long assignments.
```

### Mistake 7 — Formatting generated code

```
# ❌ WRONG: no .prettierignore entry for generated output.
# Every `prisma generate` / `graphql-codegen` run writes ITS formatting;
# Prettier rewrites it; the next generation rewrites it back. git status is
# never clean and CI flaps.

# ✅ RIGHT: the generator owns its output. Full stop.
# .prettierignore
src/generated/
prisma/generated/
*.gen.ts
*.pb.ts
**/__snapshots__/
```

```jsonc
// ✅ And exclude the same paths from ESLint, for the same reason:
export default tseslint.config(
  { ignores: ["dist/**", "coverage/**", "src/generated/**", "**/*.gen.ts"] },
  // ...
);
```

### Mistake 8 — `.prettierignore` in the wrong place for how you invoke Prettier

```bash
# ❌ WRONG: running from a package directory in a monorepo.
cd packages/api
npx prettier --check .
# The root .prettierignore is NOT read — .prettierignore resolves relative to
# the CWD. Prettier now happily checks packages/api/dist/ and fails.
```

```bash
# ✅ RIGHT, option A: always run from the repo root.
npm run format:check         # with "format:check": "prettier --check ." at root

# ✅ RIGHT, option B: pass the ignore path explicitly.
cd packages/api
npx prettier --check . --ignore-path ../../.prettierignore

# ✅ RIGHT, option C: give each package its own .prettierignore.
#    (Duplication, but unambiguous — often the pragmatic monorepo choice.)
```

---

## Practice exercises

### Exercise 1 — easy

Set up Prettier from scratch on a new TypeScript project and prove to yourself exactly what it does and does not touch.

Concretely:

1. `mkdir prettier-lab && cd prettier-lab && npm init -y && npm i -D --save-exact prettier && npm i -D typescript @types/node && npx tsc --init`.
2. Write `.prettierrc` with `semi: true`, `singleQuote: false`, `trailingComma: "all"`, `printWidth: 100`, `tabWidth: 2`, `endOfLine: "lf"`.
3. Write `.prettierignore` covering `dist/`, `node_modules/`, `coverage/`, `package-lock.json`.
4. Create `src/inventory.ts` containing, deliberately badly formatted (mixed indentation, single quotes, no semicolons, comma-delimited interface members, everything crammed onto long lines):
   - an `interface StockItem` with `sku`, `warehouseId`, `quantityOnHand`, `reservedQuantity`
   - a `type StockStatus` union of `"in_stock" | "low" | "out_of_stock" | "discontinued"`
   - a function `classifyStock(item: StockItem, lowThreshold: number): StockStatus`
   - an unused import from `node:crypto`
   - imports deliberately out of alphabetical order
5. Run `npx prettier --check src/inventory.ts` and record the exact output and exit code (`echo $?`).
6. Run `npx prettier src/inventory.ts` (no `--write`) to see the result on stdout without changing the file.
7. Run `npx prettier --write src/inventory.ts`. Now write down answers to these, verified against the file:
   - Did the interface's `,` separators become `;`? Why is that not configurable?
   - Did the unused `node:crypto` import get removed?
   - Did the imports get reordered?
   - Did any line exceed 100 characters? Which construct, and why couldn't Prettier break it?
8. Change `printWidth` to `40`, re-run `--write`, and observe what happens to the union type and the function signature. Change it back.
9. Add `// prettier-ignore` above a hand-aligned lookup table and confirm it survives a `--write`.
10. Run `npx prettier --file-info src/inventory.ts` and `npx prettier --find-config-path src/inventory.ts` and explain what each tells you.

```
// Write your code here
```

### Exercise 2 — medium

Wire Prettier and ESLint together correctly on a small Express + TypeScript service, then prove they do not conflict.

Concretely:

1. Scaffold a service with `src/server.ts`, `src/routes/orderRoutes.ts`, `src/services/orderService.ts`, `src/db/pool.ts`, and `src/generated/client.ts` (fake generated file with intentionally odd formatting).
2. Install `eslint`, `typescript-eslint`, `eslint-config-prettier`, `eslint-plugin-import`, `prettier` (pinned exactly).
3. **First, create the conflict on purpose.** Write a flat `eslint.config.js` that enables `@stylistic/indent` (or legacy `indent`), `quotes`, `semi`, and `max-len: 100` — and do NOT include `eslint-config-prettier`. Set `printWidth: 100`, `tabWidth: 2`, `singleQuote: false` in `.prettierrc`, and set the ESLint rules to the *opposite* values (4-space indent, single quotes). Then:
   - Run `npx prettier --write . && npx eslint .` and record how many errors ESLint reports on freshly-formatted code.
   - Run `npx eslint . --fix && npx prettier --check .` and record how many files Prettier now considers unformatted.
   - Write two sentences describing the loop you just created.
4. **Now fix it.** Add `eslintConfigPrettier` as the last entry in the flat config array. Re-run both commands and confirm both are clean.
5. Verify mechanically: run `npx eslint-config-prettier src/server.ts` and confirm it reports no conflicting rules.
6. Add correctness rules that Prettier can never affect: `@typescript-eslint/no-floating-promises`, `no-misused-promises`, `switch-exhaustiveness-check`, `consistent-type-imports`. Write code in `orderService.ts` that violates each one and confirm ESLint catches all four while `prettier --check` stays green. This is the point of the exercise: **format-clean and lint-clean are independent properties.**
7. Add `import/order` with `newlines-between: "always"` and alphabetisation. Scramble the imports in all four source files, run `npx prettier --write .`, confirm Prettier did nothing to the order, then run `npx eslint . --fix` and confirm the order is now correct.
8. Add `src/generated/` to both `.prettierignore` and the ESLint `ignores` array. Confirm `npx prettier --check .` passes despite `src/generated/client.ts` being badly formatted.
9. Add `package.json` scripts `format`, `format:check`, `lint`, `typecheck`, and a `verify` that chains all three. Make `verify` pass.
10. Finally, install `eslint-plugin-prettier` and switch to `plugin:prettier/recommended` in a branch. Time `npx eslint .` with and without it (`time` or `hyperfine`), and record the difference and the change in output noise. Then revert. Write three sentences on why you reverted.

```
// Write your code here
```

### Exercise 3 — hard

Adopt Prettier on a realistic multi-package repository with full enforcement — hooks, CI, blame hygiene, and a documented team decision record.

Concretely:

1. **Build the repo.** Create a workspace monorepo:
   ```
   acme/
   ├── packages/shared/src/**          (domain types, pure functions)
   ├── services/api/src/**             (Express + Postgres)
   └── services/worker/src/**          (a queue consumer)
   ```
   Seed each package with 15+ TypeScript files, formatted inconsistently on purpose — different quote styles, indentation, and semicolon usage per package, as if three people wrote them.

2. **Root config, package overrides.** Put `.prettierrc` and `.prettierignore` at the root. Give `services/api` a *different* `printWidth` two ways, and compare them:
   - (a) a nested `packages/../.prettierrc` — observe that it REPLACES the root config, not merges, and that all your other options are lost.
   - (b) an `overrides` block in the root config keyed on `services/api/**` — observe that it merges.
   Write which one you'd standardise on and why.

3. **The big reformat.** Commit the unformatted state first. Then run `npx prettier --write .` as a single commit containing nothing else. Create `.git-blame-ignore-revs` with that SHA, run `git config blame.ignoreRevsFile .git-blame-ignore-revs`, and demonstrate with `git blame src/<somefile>.ts` that the original authorship is preserved.

4. **Hooks.** Install husky v9 + lint-staged. Configure:
   - pre-commit: `npx lint-staged` running `prettier --write` then `eslint --fix --max-warnings=0` on staged TS, and `prettier --write` on staged JSON/MD/YAML.
   - pre-push: `tsc --noEmit` per package.
   Then deliberately break it: put `tsc --noEmit` into the lint-staged TS glob, commit a file, and observe that lint-staged appends filenames to `tsc` and therefore silently ignores `tsconfig.json`. Prove this by adding a `strict` violation that passes the hook. Fix it with the function-form config.

5. **Time the hook.** Measure pre-commit duration for a 1-file commit and a 40-file commit. If either exceeds 3 seconds, make it faster and document what you changed. Explain in one sentence why hook latency is a correctness concern, not a comfort concern.

6. **CI.** Write `.github/workflows/ci.yml` with four separately-named steps: format check, lint, typecheck, test. Then:
   - Push a commit with only a whitespace violation. Confirm ONLY the format step is red, and that the log names the file.
   - Push a commit with only a floating promise. Confirm ONLY the lint step is red.
   - Add `--cache --cache-strategy content` to the Prettier step plus an `actions/cache` entry, and measure the cold vs warm difference.
   - Add a changed-files-only variant of the format step and measure it against the full check.

7. **Import authority bake-off.** Implement import ordering three ways on three branches — `eslint-plugin-import`, `@ianvs/prettier-plugin-sort-imports`, and `prettier-plugin-organize-imports`. For each, record: (a) does it run on save, (b) is it enforceable in CI, (c) does it remove unused imports, (d) does it move side-effect imports (test this with a `import "./bootstrap/telemetry.js";` that MUST run first). Then deliberately enable two at once and document the fight.

8. **Version pinning.** Pin Prettier exactly. Then bump it to a different minor version, run `--write`, and count how many files changed. Write the upgrade runbook your team would follow: what to run, what to commit, and what to add to `.git-blame-ignore-revs`.

9. **Editor parity.** Commit `.vscode/settings.json` and `.vscode/extensions.json`. Verify that a clean checkout with the extension installed formats on save using the repo's pinned Prettier (`prettier.prettierPath`) and not the extension's bundled copy — prove it by pinning an older Prettier and observing the output difference.

10. **Write `docs/CODE_STYLE.md`** containing:
    - The team's decision on `semi` and `trailingComma`, with the one-paragraph rationale each.
    - The rule "formatting is never a review comment" and the mechanism that makes it true.
    - A symptom → cause → fix table with at least eight entries drawn from failures you actually hit in steps 1–9.
    - The upgrade runbook from step 8.

```
// Write your code here
```

---

## Quick reference cheat sheet

```bash
# ── Install ─────────────────────────────────────────────────────────────────
npm i -D --save-exact prettier
npm i -D eslint-config-prettier          # turns OFF conflicting ESLint rules
npm i -D husky lint-staged               # pre-commit enforcement

# ── Commands ────────────────────────────────────────────────────────────────
npx prettier --write .                   # format everything in place
npx prettier --check .                   # CI gate; exit 1 if unformatted
npx prettier -l .                        # list differing filenames only
npx prettier src/x.ts                    # print formatted result to stdout
npx prettier --check . --cache           # use the content-hash cache
npx prettier --file-info src/x.ts        # parser + ignored status
npx prettier --find-config-path src/x.ts # which config applies here?
npx prettier --support-info              # all options, all languages
npx eslint-config-prettier src/x.ts      # report any remaining rule conflicts
```

```jsonc
// ── .prettierrc — the options worth setting ─────────────────────────────────
"semi": true                 // ; at statement ends (default true)
"singleQuote": false         // "double" quotes (default false)
"trailingComma": "all"       // trailing commas everywhere (default in v3) ← use this
"printWidth": 100            // SOFT target line length (default 80)
"tabWidth": 2                // spaces per indent (default 2)
"useTabs": false             // spaces not tabs (default false)
"arrowParens": "always"      // (x) => x (default always)
"bracketSpacing": true       // { a: 1 } (default true)
"quoteProps": "as-needed"    // quote only keys that need it (default)
"endOfLine": "lf"            // normalise CRLF → LF (default lf)
"objectWrap": "preserve"     // honour your newline after { (v3.5+, default)
"embeddedLanguageFormatting": "auto"   // format CSS/GraphQL in template literals
"plugins": []                // plugin load order matters
"overrides": []              // per-glob option changes — MERGES with the base
```

```
# ── .prettierignore — .gitignore syntax; resolved from the CWD ──────────────
dist/  build/  coverage/  node_modules/
src/generated/  *.gen.ts  **/__snapshots__/
package-lock.json  pnpm-lock.yaml
```

```jsonc
// ── .vscode/settings.json ───────────────────────────────────────────────────
"editor.defaultFormatter": "esbenp.prettier-vscode"   // extension ID, not a name
"editor.formatOnSave": true
"[typescript]": { "editor.defaultFormatter": "esbenp.prettier-vscode" }
"prettier.requireConfig": true                        // opt-in per repo
"prettier.prettierPath": "./node_modules/prettier"    // use the pinned version
"editor.codeActionsOnSave": { "source.fixAll.eslint": "explicit" }
```

```js
// ── ESLint integration ──────────────────────────────────────────────────────
// FLAT (ESLint 9+):    export default tseslint.config(..., prettierConfig);  ← last
// LEGACY (.eslintrc):  extends: [..., "prettier"]                            ← last
// Do NOT default to eslint-plugin-prettier / "plugin:prettier/recommended".
```

```jsonc
// ── lint-staged + husky ─────────────────────────────────────────────────────
"lint-staged": {
  "*.{ts,tsx}": ["prettier --write", "eslint --fix --max-warnings=0"],
  "*.{json,md,yml,yaml,css}": ["prettier --write"]
}
// .husky/pre-commit  →  npx lint-staged
// .husky/pre-push    →  npm run typecheck      (too slow for pre-commit)
// Skip once:  git commit --no-verify   |   HUSKY=0 git commit
```

```ts
// ── Escape hatches ──────────────────────────────────────────────────────────
// prettier-ignore          → skips the NEXT node only
{/* prettier-ignore */}     → the JSX form
// (no file-level ignore comment exists — use .prettierignore)
```

| Prettier DOES | Prettier does NOT |
|---|---|
| Line breaking / wrapping | Sort or organise imports |
| Indentation | Remove unused imports or variables |
| Quote normalisation | Rename anything |
| Semicolon insert/remove | Enforce naming conventions |
| Trailing commas | Catch bugs of any kind |
| Spacing and delimiters | Type-check (never runs tsc) |
| Collapse 2+ blank lines to 1 | Guarantee a max line length |
| Format embedded CSS/GraphQL | Align values vertically |

| Symptom | Most likely cause | Fix |
|---|---|---|
| Format-on-save does nothing | No/wrong `defaultFormatter` | Set `"esbenp.prettier-vscode"` + per-language pins |
| Editor and CI disagree | Extension's bundled Prettier ≠ pinned one | `"prettier.prettierPath": "./node_modules/prettier"` |
| Save-fight: file flips back and forth | Two formatters, or two import sorters | Pick one authority per concern |
| Lint errors about indentation | `eslint-config-prettier` missing or not last | Add it as the LAST config entry |
| Every PR reformats the world | Unpinned Prettier version | `npm i -D --save-exact prettier` |
| CI format step never fails | Using `--write` instead of `--check` | Use `--check` |
| `dist/` gets formatted in a monorepo | `.prettierignore` not at the CWD | Run from root, or `--ignore-path` |
| Generated files churn every build | Missing `.prettierignore` entry | Ignore `src/generated/`, `*.gen.ts` |
| Line still over `printWidth` | Long string / identifier — unbreakable | Extract to a constant; `printWidth` is a target |
| `git blame` says "chore: prettier" | Bulk reformat commit | `.git-blame-ignore-revs` + `blame.ignoreRevsFile` |
| Nested config lost half the options | Configs REPLACE, don't merge | Use root `overrides` instead |
| Hook typecheck passes but CI fails | lint-staged passed files to `tsc` | Function form: `() => "tsc --noEmit -p tsconfig.json"` |

---

## Connected topics

- **03 — tsconfig in depth** — `noUnusedLocals` / `noUnusedParameters` do the unused-code cleanup Prettier explicitly won't, and `strict` is the correctness layer beneath both formatting and linting.
- **04 — TypeScript Node.js project setup** — where the `format`, `lint`, `typecheck`, and `verify` npm scripts live, and how they compose into one command.
- **74 — ESLint with TypeScript** — the other half of this story: `eslint-config-prettier`, correctness rules, `import/order`, and why the two tools stay separate.
- **75 — Debugging TypeScript in VS Code** — the same `.vscode/settings.json` and committed-editor-config discipline, applied to `launch.json`.
- **76 — Building for production** — CI pipeline shape; the format check is the cheapest gate and belongs before the expensive ones.
- **77 — Monorepo basics with TypeScript** — config discovery across packages, the `.prettierignore`-relative-to-CWD trap, and per-package overrides.
