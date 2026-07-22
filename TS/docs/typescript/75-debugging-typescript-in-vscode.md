# 75 — Debugging TypeScript in VS Code

## What is this?

**Debugging TypeScript** means running your program under Node's inspector protocol while VS Code maps the *running JavaScript* back to the *TypeScript you wrote* — so you set a breakpoint on line 42 of `src/services/userService.ts`, and it actually stops there, with `userId` and `requestBody` visible in the Variables pane.

The whole thing rests on one artifact: the **source map**. A source map is a JSON file (or an inline base64 comment) that says "character 118 of `dist/services/userService.js` came from line 42, column 6 of `src/services/userService.ts`".

```
  src/services/userService.ts          ← what you wrote, what you breakpoint on
            │  tsc / tsx / esbuild
            ▼
  dist/services/userService.js         ← what Node actually executes
  dist/services/userService.js.map     ← the translation table
            │  Node --inspect (CDP)
            ▼
  VS Code debugger                     ← reads the map, shows you TS
```

There are three moving parts you must get right, and debugging fails if *any one* is wrong:

1. **The compiler emits a correct source map** (`sourceMap: true` in `tsconfig.json`).
2. **VS Code can find the emitted JS** (`outFiles` in `launch.json`).
3. **The paths inside the map resolve on disk** (`sourceRoot`, `rootDir`, and where you run from).

---

## Why does it matter?

Because `console.log` debugging in a backend service is a slow, lossy way to answer questions you could answer in ten seconds with a breakpoint:

- A webhook handler throws `TypeError: Cannot read properties of undefined (reading 'email')` — you need to see the actual shape of `requestBody` at that moment, not a truncated `JSON.stringify`.
- A Prisma/Knex query returns rows in the wrong order and you want to step through the mapping loop and inspect each `userId`.
- An `authToken` fails verification in prod-like config and you want to inspect the decoded claims and the clock skew.
- A Jest test fails only in CI ordering and you want to stop *inside* the failing assertion with the full closure visible.

And there's a second, quieter reason: **stack traces**. Without source maps wired up at runtime, a production crash gives you `at dist/services/orderService.js:1:4821` — which, after bundling, is useless. With them, you get `at createOrder (src/services/orderService.ts:88:11)`.

TypeScript adds a compile step between you and the runtime. Debugging setup is how you delete that gap again.

---

## The JavaScript way vs the TypeScript way

```js
// ── Plain JavaScript / Node — debugging is trivially direct ──────────────────

// src/server.js  ← this exact file is what Node loads and runs.
function handleCreateUser(requestBody) {
  const email = requestBody.profile.email;   // ← put a breakpoint here
  return { email };
}

// .vscode/launch.json — literally 4 lines of meaningful config:
// { "type": "node", "request": "launch", "program": "${workspaceFolder}/src/server.js" }
//
// Breakpoint binds instantly. Line numbers match. Variables have the names you wrote.
// Stack trace says: at handleCreateUser (/app/src/server.js:3:34)   ✅ correct file, correct line
```

Now the same thing after you add TypeScript, **without** configuring anything:

```js
// ── TypeScript, misconfigured — the gap opens up ─────────────────────────────

// You breakpoint src/server.ts line 3.
// Node is running dist/server.js — a DIFFERENT file it has never heard of.
//
// VS Code shows:  ⭘  "Unbound breakpoint"  (hollow grey circle, not solid red)
// The program runs straight past it and exits.
//
// And the crash you were chasing prints:
//   TypeError: Cannot read properties of undefined (reading 'email')
//       at handleCreateUser (/app/dist/server.js:14:41)   ❌ a file you never wrote
//       at /app/dist/routes/index.js:88:22                ❌ line 88 of what, exactly?
//
// You now debug by bisecting console.logs into a file you don't edit. This is the
// entire reason people say "TypeScript makes debugging harder". It doesn't. Config does.
```

```ts
// ── The TypeScript way — three settings and the gap closes completely ────────

// 1. tsconfig.json  → tell the compiler to emit the translation table
//    { "compilerOptions": { "sourceMap": true, "outDir": "dist", "rootDir": "src" } }

// 2. .vscode/launch.json → tell VS Code where the emitted JS lives
//    { "outFiles": ["${workspaceFolder}/dist/**/*.js"] }

// 3. src/server.ts → unchanged. You just write TypeScript.
function handleCreateUser(requestBody: CreateUserRequest): { email: string } {
  const email = requestBody.profile.email;   // ← breakpoint binds. Solid red dot. It stops.
  return { email };
}

// Breakpoint: ✅ binds on the .ts line
// Variables pane: ✅ shows `requestBody` with the shape you expect
// Step Into: ✅ steps into src/services/userService.ts, not dist/
// Stack trace: ✅ at handleCreateUser (src/server.ts:3:34)
//
// You are back to the JavaScript experience — with types on top.
```

The revelation: **TypeScript debugging is not "different" debugging.** It is normal Node debugging plus one JSON file that translates coordinates. Once that file exists and both VS Code and Node can find it, everything you knew about `debugger`, breakpoints, watch expressions, and stack traces works identically.

---

## Syntax

```jsonc
// ── tsconfig.json — the source map options ───────────────────────────────────
{
  "compilerOptions": {
    "outDir": "dist",              // where the .js goes
    "rootDir": "src",              // where the .ts comes from — anchors relative map paths
    "sourceMap": true,             // emit dist/x.js.map  (separate file) — the normal choice
    // "inlineSourceMap": true,    // embed the map as a base64 comment IN the .js (mutually exclusive with sourceMap)
    // "inlineSources": true,      // also embed the ORIGINAL .ts text inside the map
    // "sourceRoot": "/",          // prefix prepended to source paths inside the map — rarely needed
    // "mapRoot": "https://cdn/",  // where the .map files live if not next to the .js — rarer still
    "declarationMap": true         // maps .d.ts back to .ts — powers go-to-definition (see 77)
  }
}
```

```jsonc
// ── .vscode/launch.json — minimal launch of compiled output ──────────────────
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",                                   // Node debug adapter (built into VS Code)
      "request": "launch",                              // "launch" = start it | "attach" = connect to running
      "name": "Debug API (compiled)",                   // shows in the Run and Debug dropdown
      "program": "${workspaceFolder}/dist/server.js",   // the JS entry point Node executes
      "preLaunchTask": "npm: build",                    // compile first, so dist/ is fresh
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],  // where to look for source maps — CRITICAL
      "sourceMaps": true,                               // default true; here for explicitness
      "skipFiles": ["<node_internals>/**"],             // don't step into Node's own C++/JS internals
      "console": "integratedTerminal",                  // real TTY — stdin works, colours work
      "env": { "NODE_ENV": "development" }              // env vars for the debuggee
    }
  ]
}
```

```bash
# ── Node inspector flags — what the debugger actually connects to ────────────
node --inspect dist/server.js          # open inspector on 127.0.0.1:9229, DON'T wait
node --inspect-brk dist/server.js      # open inspector AND pause on the first line
node --inspect=0.0.0.0:9229 dist/...   # listen on all interfaces (Docker) — never do this in prod
node --inspect-port=9230 dist/...      # different port, e.g. two services at once
```

---

## How it works — concept by concept

### Concept 1 — What a source map actually contains

A `.js.map` is plain JSON. Understanding its four load-bearing fields makes every "breakpoint won't bind" bug diagnosable in thirty seconds.

```jsonc
// dist/services/userService.js.map (formatted; real ones are one line)
{
  "version": 3,                                  // source map spec version — always 3
  "file": "userService.js",                      // the generated file this map describes
  "sourceRoot": "",                              // prefix prepended to every entry in "sources"
  "sources": ["../../src/services/userService.ts"], // RELATIVE to the .map file's own location
  "names": ["userId", "findUserById", "requestBody"], // original identifier names
  "mappings": "AAAA,SAAgB,YAAhB,CAA6B..."      // base64 VLQ position deltas — the real payload
  // "sourcesContent": ["export async function findUserById..."]  // present if inlineSources: true
}
```

And at the bottom of `dist/services/userService.js`:

```js
//# sourceMappingURL=userService.js.map
```

That comment is how *any* tool — VS Code, Node, Sentry, `source-map-support` — finds the map. Three rules follow directly from this structure:

```js
// RULE 1: the comment must survive. Minifiers, bundlers and "strip comments" post-processing
//         delete it. No comment → no map → no breakpoints, no matter how good the map is.

// RULE 2: "sources" paths are resolved relative to the .map file on disk.
//         Move dist/ somewhere else without moving src/, and "../../src/..." points at nothing.
//         This is exactly why Docker images that copy only dist/ show dist/*.js in stack traces.

// RULE 3: "mappings" encodes positions, not semantics. If you edit src/ after building,
//         the map is stale: your breakpoint on line 42 binds to whatever line 42 *used to be*.
//         Symptom: the debugger stops on a visibly wrong line. Fix: rebuild.
```

### Concept 2 — `sourceMap` vs `inlineSourceMap` vs `inlineSources`

Three options, three different trade-offs. They are frequently confused.

```jsonc
// ── Option A: sourceMap: true  (separate .map file) ─────────────────────────
// Emits: dist/server.js  +  dist/server.js.map
// ✅ The shipped .js stays small — the map is only downloaded/read when a debugger asks.
// ✅ You can ship the .js and withhold the .map (or upload it to Sentry only).
// ❌ Two files to keep together; delete/move one and debugging silently dies.
{ "compilerOptions": { "sourceMap": true } }

// ── Option B: inlineSourceMap: true  (base64 comment inside the .js) ────────
// Emits: dist/server.js only, ending with
//        //# sourceMappingURL=data:application/json;base64,eyJ2ZXJzaW9u...
// ✅ Impossible to lose the map — it travels with the code. Great for Docker, Lambda, CI artifacts.
// ✅ No extra file resolution → fewer "map not found" failures.
// ❌ Bloats every .js file substantially (often 2–4×). Slower cold start on big services.
// ⚠️  MUTUALLY EXCLUSIVE with sourceMap — setting both is a tsc error.
{ "compilerOptions": { "inlineSourceMap": true } }

// ── Option C: inlineSources: true  (original .ts TEXT inside the map) ───────
// Works with either A or B. Puts "sourcesContent" into the map.
// ✅ The debugger can show your original TypeScript even if src/ is NOT on disk.
//    This is the option that makes debugging inside a slim Docker image work.
// ❌ Your source code is now embedded in shipped artifacts. Fine for a private service;
//    think twice for a published npm package or a client-side bundle.
{ "compilerOptions": { "sourceMap": true, "inlineSources": true } }
```

Practical picks:

| Scenario | Setting |
|---|---|
| Local dev of a service | `sourceMap: true` |
| Docker image, src/ not copied | `sourceMap: true, inlineSources: true` |
| Serverless / single-file artifact | `inlineSourceMap: true, inlineSources: true` |
| Published npm library | `sourceMap: true, declarationMap: true`, ship `src/` in `files` |

### Concept 3 — `sourceRoot`: what it is and why you usually shouldn't touch it

`sourceRoot` is a string prepended to every path in `sources` when a consumer resolves them.

```jsonc
// Without sourceRoot — the default, and almost always correct:
// map lives at  dist/services/userService.js.map
// sources:      ["../../src/services/userService.ts"]
// resolved:     <repo>/src/services/userService.ts    ✅ found
{ "compilerOptions": { "sourceMap": true } }

// With sourceRoot — the paths become sourceRoot + sources entry:
{ "compilerOptions": { "sourceMap": true, "sourceRoot": "/app/src" } }
// sources become effectively /app/src/../../src/... which is usually WRONG on your laptop
// but can be exactly right inside a container where the code lives at /app.
```

The single legitimate use: **the path at build time differs from the path at debug time.** Building in `/builds/ci/xyz` but debugging in a container at `/app`:

```jsonc
// tsconfig.build.json used in the Docker build stage:
{ "compilerOptions": { "sourceMap": true, "sourceRoot": "/app/src" } }
```

Otherwise prefer the VS Code-side fix, which doesn't pollute your artifacts:

```jsonc
// launch.json — remap paths at debug time instead of at build time
{
  "type": "node",
  "request": "attach",
  "name": "Attach to container",
  "port": 9229,
  "address": "localhost",
  "localRoot": "${workspaceFolder}",   // where the source is on YOUR machine
  "remoteRoot": "/app",                // where it was when it was compiled
  "outFiles": ["${workspaceFolder}/dist/**/*.js"],
  "skipFiles": ["<node_internals>/**"]
}
```

### Concept 4 — `outFiles`: the option that fixes 80% of unbound breakpoints

`outFiles` is a glob list telling VS Code **which generated files to scan for source maps before the program starts**. VS Code reads their maps, builds a reverse index (`.ts` line → `.js` line), and *pre-places* breakpoints. Without it, VS Code only learns about a map when the file is loaded — by which time your breakpoint on module top-level code has already been missed.

```jsonc
// ✅ Correct — points at the compiled output, includes a recursive glob
"outFiles": ["${workspaceFolder}/dist/**/*.js"]

// ✅ Multiple output roots (monorepo — see 77)
"outFiles": [
  "${workspaceFolder}/packages/*/dist/**/*.js",
  "${workspaceFolder}/services/api/dist/**/*.js"
]

// ✅ Exclude noise — a leading ! negates. Speeds up startup on big trees.
"outFiles": [
  "${workspaceFolder}/dist/**/*.js",
  "!**/node_modules/**"
]

// ❌ Points at SOURCE, not output — VS Code finds no maps
"outFiles": ["${workspaceFolder}/src/**/*.ts"]

// ❌ Non-recursive — misses dist/services/, dist/routes/, dist/middleware/
"outFiles": ["${workspaceFolder}/dist/*.js"]

// ❌ Wrong root — a leftover from before you renamed dist → build
"outFiles": ["${workspaceFolder}/out/**/*.js"]
```

### Concept 5 — `skipFiles`: making step-through usable

By default, "Step Into" will happily drop you into `node:internal/modules/cjs/loader`, Express's `Layer.handle`, or 400 lines of `node_modules/pg/lib/client.js`. `skipFiles` blacklists those so stepping goes where you care.

```jsonc
"skipFiles": [
  "<node_internals>/**",                    // Node's own internals — always include this
  "${workspaceFolder}/node_modules/**",     // all third-party code
  "**/async_hooks.js",                      // async wrappers that pollute call stacks
  "!${workspaceFolder}/node_modules/@acme/**" // ...except our own workspace packages (! un-skips)
]
```

Skipped frames are also collapsed in the Call Stack pane behind a "Show N more frames" link — so a 30-frame Express stack becomes the 4 frames that are yours. Note the `!` negation is evaluated in order, so put un-skips *after* the broad skip.

### Concept 6 — `--inspect` vs `--inspect-brk`

Both open the V8 Inspector (Chrome DevTools Protocol) on a WebSocket, default `127.0.0.1:9229`.

```bash
node --inspect dist/server.js
# Prints: Debugger listening on ws://127.0.0.1:9229/6f1a...
#         For help, see: https://nodejs.org/en/docs/inspector
# Execution starts IMMEDIATELY. Anything that runs during module load — config parsing,
# DI container wiring, a top-level `await migrate()` — happens before you can attach.

node --inspect-brk dist/server.js
# Same, but V8 pauses BEFORE executing the first line of the entry module.
# Use this when the bug is in startup: env validation, DB pool creation, route registration.
```

The rule of thumb:

```
Bug happens while handling a request  →  --inspect       (attach whenever, then hit the endpoint)
Bug happens during boot               →  --inspect-brk   (attach first, then let it run)
Long-running server already up        →  --inspect       + an "attach" launch config
Short script that exits in 200ms      →  --inspect-brk   (otherwise it's over before you connect)
```

Security note that matters for backend work: `--inspect` exposes **arbitrary code execution** to anything that can reach the port. Never bind it to `0.0.0.0` on a public host, never enable it in production, and if you must debug in a container, forward the port over SSH rather than publishing it.

### Concept 7 — How the debugger binds a breakpoint (and why it fails)

The sequence, in order — each step is a place it can break:

```
1. You click the gutter at src/services/orderService.ts:88.
2. VS Code scans `outFiles` globs, reads every .js.map it finds.
3. It searches the maps for one whose `sources` includes .../src/services/orderService.ts.
4. It translates line 88 → dist/services/orderService.js line 141, column 12.
5. It sends Debugger.setBreakpointByUrl(file:///.../dist/services/orderService.js, 141, 12) over CDP.
6. V8 replies with an "actual location". If V8 relocated it, VS Code moves the dot.
7. Dot goes SOLID RED = bound. HOLLOW GREY = step 3, 4, or 5 failed.
```

Diagnosing from the symptom:

| Symptom | Failing step | Fix |
|---|---|---|
| Hollow grey dot, never binds | 2 or 3 | `outFiles` wrong, or `sourceMap` not enabled — rebuild and check `dist/*.js.map` exists |
| Binds, but stops on the wrong line | stale map | `dist/` is out of date — add `preLaunchTask` to build |
| Binds, but stops in `dist/*.js` not `.ts` | 3 | `sources` paths don't resolve — check `rootDir`/`sourceRoot`/`localRoot` |
| Binds only after the first request | 2 | breakpoint set after load; add `outFiles`, or use `--inspect-brk` |
| Works for some files, not others | glob | non-recursive `outFiles`, or the file is excluded from the tsconfig `include` |

---

## Example 1 — basic

A minimal Express service you can breakpoint end to end.

```jsonc
// ── tsconfig.json ───────────────────────────────────────────────────────────
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",                 // anchors the relative paths written into the maps
    "outDir": "dist",                 // where tsc writes .js and .js.map
    "sourceMap": true,                // ← the one setting that makes debugging possible
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

```jsonc
// ── package.json (excerpt) ──────────────────────────────────────────────────
{
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js",
    "debug": "node --inspect-brk dist/server.js"   // CLI equivalent of the launch config
  }
}
```

```jsonc
// ── .vscode/tasks.json — so VS Code can build before launching ──────────────
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "npm",
      "script": "build",
      "problemMatcher": ["$tsc"],   // surfaces tsc errors in the Problems pane
      "group": "build",
      "label": "npm: build"         // must match `preLaunchTask` exactly
    }
  ]
}
```

```jsonc
// ── .vscode/launch.json ─────────────────────────────────────────────────────
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug API server",
      "program": "${workspaceFolder}/dist/server.js",  // Node runs the COMPILED entry point
      "preLaunchTask": "npm: build",                   // ensures dist/ matches src/ before we start
      "outFiles": ["${workspaceFolder}/dist/**/*.js"], // where VS Code hunts for .js.map files
      "skipFiles": ["<node_internals>/**"],
      "console": "integratedTerminal",
      "env": {
        "NODE_ENV": "development",
        "PORT": "3000",
        "DATABASE_URL": "postgres://localhost:5432/app_dev"
      }
    }
  ]
}
```

```ts
// ── src/server.ts ───────────────────────────────────────────────────────────
import express, { type Request, type Response } from "express";
import { findUserById } from "./services/userService.js";

const app = express();
app.use(express.json());

app.get("/users/:userId", async (req: Request, res: Response) => {
  const userId = Number(req.params.userId);   // ← breakpoint here: inspect req.params
  if (!Number.isInteger(userId) || userId <= 0) {
    return res.status(400).json({ error: "INVALID_USER_ID" });
  }

  const user = await findUserById(userId);    // ← F11 (Step Into) lands in userService.ts, not dist/
  if (!user) return res.status(404).json({ error: "NOT_FOUND" });

  return res.json({ data: user });
});

const port = Number(process.env.PORT ?? 3000);
app.listen(port, () => {
  console.log(`listening on ${port}`);        // ← breakpoint here binds only with outFiles set
});
```

```ts
// ── src/services/userService.ts ─────────────────────────────────────────────
export interface User {
  userId: number;
  email: string;
  displayName: string;
}

const usersById = new Map<number, User>([
  [1, { userId: 1, email: "ada@example.com", displayName: "Ada" }],
]);

export async function findUserById(userId: number): Promise<User | null> {
  // Breakpoint here. Variables pane shows `userId: 1` with the TS name preserved
  // because tsc kept identifier names and the map's "names" array records them.
  const user = usersById.get(userId) ?? null;
  return user;
}
```

Run it: open the Run and Debug panel → pick "Debug API server" → F5. `curl localhost:3000/users/1` and you stop on line 8 of `server.ts`.

---

## Example 2 — real world backend use case

A full `launch.json` for a real service: compiled output, `tsx` dev loop, attach-to-running, attach-to-Docker, Jest, and Vitest — plus `source-map-support` for readable production stack traces.

```jsonc
// ── .vscode/launch.json — the complete working set ───────────────────────────
{
  "version": "0.2.0",
  "configurations": [
    // ─────────────────────────────────────────────────────────────────────────
    // 1. Launch the COMPILED build. Slowest to start, closest to production.
    //    Use when a bug only reproduces against real emitted output.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "launch",
      "name": "API: launch compiled",
      "program": "${workspaceFolder}/dist/server.js",
      "preLaunchTask": "npm: build",
      "outFiles": ["${workspaceFolder}/dist/**/*.js", "!**/node_modules/**"],
      "skipFiles": ["<node_internals>/**", "${workspaceFolder}/node_modules/**"],
      "console": "integratedTerminal",
      "envFile": "${workspaceFolder}/.env.development",   // loads KEY=value pairs
      "env": { "NODE_ENV": "development" }                // `env` wins over `envFile`
    },

    // ─────────────────────────────────────────────────────────────────────────
    // 2. Launch through tsx — no build step, sub-second start.
    //    tsx emits INLINE source maps, so no `outFiles` is needed; the maps
    //    ride inside the in-memory JS that tsx hands to Node.
    //    NOTE: tsx transpiles only — it does NOT type-check. See 76.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "launch",
      "name": "API: launch via tsx",
      "runtimeExecutable": "tsx",                 // resolved from node_modules/.bin
      "args": ["${workspaceFolder}/src/server.ts"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**", "**/node_modules/tsx/**"],
      "env": { "NODE_ENV": "development", "TSX_TSCONFIG_PATH": "./tsconfig.json" }
    },

    // ─────────────────────────────────────────────────────────────────────────
    // 2b. Same idea with ts-node (CommonJS projects). Requires `sourceMap: true`
    //     in tsconfig — ts-node reads it and feeds maps to Node's inspector.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "launch",
      "name": "API: launch via ts-node",
      "runtimeArgs": ["-r", "ts-node/register"],  // preload the compiler hook
      "program": "${workspaceFolder}/src/server.ts",
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**", "**/node_modules/ts-node/**"],
      "env": {
        "NODE_ENV": "development",
        "TS_NODE_TRANSPILE_ONLY": "true",         // skip type-check for a fast start
        "TS_NODE_PROJECT": "tsconfig.json"
      }
    },

    // ─────────────────────────────────────────────────────────────────────────
    // 3. ATTACH to a process you started yourself:
    //      node --inspect dist/server.js
    //      npx tsx --inspect src/server.ts
    //      npm run dev   (if the script already passes --inspect)
    //    Use when the server is managed by nodemon/docker-compose/turbo.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "attach",
      "name": "API: attach to :9229",
      "port": 9229,
      "address": "127.0.0.1",
      "restart": true,                            // reattach after nodemon restarts the process
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "skipFiles": ["<node_internals>/**"]
    },

    // ─────────────────────────────────────────────────────────────────────────
    // 4. ATTACH into a Docker container.
    //    Container CMD:  node --inspect=0.0.0.0:9229 dist/server.js
    //    compose ports:  "9229:9229"
    //    Paths differ between host and container → localRoot/remoteRoot remap them.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "attach",
      "name": "API: attach to Docker",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}",          // /Users/me/repos/api  (host)
      "remoteRoot": "/app",                       // /app                 (container)
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "skipFiles": ["<node_internals>/**"],
      "restart": { "delay": 1000, "maxAttempts": 10 }
    },

    // ─────────────────────────────────────────────────────────────────────────
    // 5. Debug the CURRENTLY OPEN Jest test file.
    //    --runInBand is mandatory: workers are child processes the debugger
    //    would otherwise never see.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "launch",
      "name": "Jest: debug current file",
      "program": "${workspaceFolder}/node_modules/jest/bin/jest.js",
      "args": [
        "--runInBand",                            // single process — required for breakpoints
        "--no-cache",                             // stale transform cache = stale source maps
        "--testTimeout=600000",                   // don't fail while you sit on a breakpoint
        "${relativeFile}"                         // only the file in the active editor
      ],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen",
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"],
      "env": { "NODE_ENV": "test", "NODE_OPTIONS": "--experimental-vm-modules" }
    },

    // ─────────────────────────────────────────────────────────────────────────
    // 6. Debug the CURRENTLY OPEN Vitest test file.
    //    Vitest needs threads disabled for the same reason Jest needs runInBand.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "launch",
      "name": "Vitest: debug current file",
      "program": "${workspaceFolder}/node_modules/vitest/vitest.mjs",
      "args": ["run", "--pool=forks", "--poolOptions.forks.singleFork", "${relativeFile}"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "smartStep": true,                          // skip generated code with no useful mapping
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"],
      "env": { "NODE_ENV": "test" }
    },

    // ─────────────────────────────────────────────────────────────────────────
    // 7. Debug a one-off migration/seed script with args.
    // ─────────────────────────────────────────────────────────────────────────
    {
      "type": "node",
      "request": "launch",
      "name": "Script: backfill user emails",
      "runtimeExecutable": "tsx",
      "args": ["${workspaceFolder}/scripts/backfillUserEmails.ts", "--batch-size=500", "--dry-run"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "envFile": "${workspaceFolder}/.env.development"
    }
  ],

  // Run two configs together — e.g. API + worker, both debuggable.
  "compounds": [
    {
      "name": "Full stack: API + worker",
      "configurations": ["API: launch via tsx", "Worker: launch via tsx"],
      "stopAll": true                             // stopping one stops the other
    }
  ]
}
```

```ts
// ── src/bootstrap/sourceMaps.ts ─────────────────────────────────────────────
// Readable stack traces at RUNTIME (not just in the debugger).
//
// Node has supported this natively since v12 behind a flag, and since v20 it is
// stable: run with `node --enable-source-maps dist/server.js`, or set
// NODE_OPTIONS=--enable-source-maps. Prefer that. Use the `source-map-support`
// package only if you're on an older runtime or need its extra hooks.
//
//   npm i source-map-support
//   npm i -D @types/source-map-support
import "source-map-support/register.js";

// With it installed, this:
//   Error: user not found
//       at findUserById (/app/dist/services/userService.js:14:11)
// becomes this:
//   Error: user not found
//       at findUserById (/app/src/services/userService.ts:22:9)
//
// Requirements — all three, or it silently no-ops:
//   1. sourceMap: true (or inlineSourceMap) at build time
//   2. the .js.map files present next to the .js in the image
//   3. the .ts files reachable, OR inlineSources: true so the map carries them

// Increase captured frames so async chains aren't truncated:
Error.stackTraceLimit = 50;

// Structured, source-mapped logging of unhandled failures:
process.on("unhandledRejection", (reason: unknown) => {
  const err = reason instanceof Error ? reason : new Error(String(reason));
  console.error(JSON.stringify({
    level: "fatal",
    event: "unhandledRejection",
    message: err.message,
    stack: err.stack,          // ← mapped back to .ts thanks to the register import above
  }));
  process.exit(1);
});
```

```ts
// ── src/server.ts — import the bootstrap FIRST, before anything else ────────
import "./bootstrap/sourceMaps.js";   // must be the very first import: it patches
                                      // Error.prepareStackTrace, and only frames
                                      // captured AFTER that patch get remapped.
import { createApp } from "./app.js";
import { loadConfig } from "./config.js";

const config = loadConfig(process.env);
createApp(config).listen(config.port);
```

```jsonc
// ── Dockerfile fragment for a debuggable dev container ──────────────────────
// docker-compose.yml:
//   services:
//     api:
//       command: node --enable-source-maps --inspect=0.0.0.0:9229 dist/server.js
//       ports: ["3000:3000", "9229:9229"]
//   ⚠️ 0.0.0.0 is acceptable ONLY on a local dev machine. Never in a deployed env.
```

---

## Going deeper

### `smartStep` — the option nobody enables and everybody needs

```jsonc
"smartStep": true
```

When stepping, VS Code skips any frame whose position has **no meaningful mapping back to source**. In TypeScript this matters because downlevel emit generates helper code you never wrote:

- `__awaiter` / `__generator` when `target` is ES5/ES2015 and you use `async`
- `__decorate` / `__metadata` with decorators (NestJS, TypeORM)
- `__spreadArray`, `__rest`, `__extends` from the tslib helpers

Without `smartStep`, F11 through an `await` in an ES5 build drops you into generator state-machine internals. With it, you land on the next line of *your* code. Note that `target: ES2022` removes most of this at the source, which is the better fix.

### `${relativeFile}` and friends — the variable that makes one config serve every test

```
${workspaceFolder}        /Users/me/repos/api
${file}                   /Users/me/repos/api/src/services/userService.test.ts
${relativeFile}           src/services/userService.test.ts
${relativeFileDirname}    src/services
${fileBasenameNoExtension} userService.test
${env:DATABASE_URL}       reads from VS Code's environment
${input:targetUserId}     prompts you at launch (define under "inputs")
```

An input-prompt config, useful for debugging a specific record:

```jsonc
{
  "configurations": [{
    "type": "node", "request": "launch", "name": "Reprocess one order",
    "runtimeExecutable": "tsx",
    "args": ["${workspaceFolder}/scripts/reprocessOrder.ts", "--orderId=${input:orderId}"]
  }],
  "inputs": [
    { "id": "orderId", "type": "promptString", "description": "Order ID to reprocess" }
  ]
}
```

### Auto Attach — debugging without ever writing a launch.json

VS Code's **Auto Attach** (Command Palette → "Debug: Toggle Auto Attach") patches `NODE_OPTIONS` in the integrated terminal so *any* Node process you start there is debuggable. Three modes:

```
"smart"        attach only to scripts NOT in node_modules  ← best default
"always"       attach to every node process (noisy: attaches to npm itself, eslint, tsc)
"onlyWithFlag" attach only when --inspect / --inspect-brk is passed explicitly
```

With `smart`, you type `npm run dev` in the terminal, set a breakpoint, and it binds — no config file. The catch: it relies on `NODE_OPTIONS` inheritance, so it does **not** work through Docker, SSH, or process managers that scrub the environment.

### Conditional, hit-count, and logpoint breakpoints

Right-click the gutter instead of left-clicking. This is where debugging beats `console.log` decisively.

```
Expression breakpoint:  userId === 4711 && requestBody.plan === "enterprise"
                        → stop only for the one tenant that reproduces the bug

Hit count:              >100          stop after 100 hits
                        %50           stop every 50th hit — sampling a hot loop

Logpoint:               Processing {userId} with {requestBody.items.length} items
                        → prints without stopping, and WITHOUT editing/redeploying code.
                        Braces interpolate expressions. This is a console.log you can
                        add to a running process and remove instantly.
```

Caveat: conditional breakpoints evaluate the expression in the debuggee on every hit. On a path executing 10⁵ times, that's genuinely slow — use a hit count or a narrower breakpoint location instead.

### Why `tsx` breakpoints "just work" but `ts-node` ones sometimes don't

`tsx` (esbuild-based) always produces inline source maps and installs them via Node's loader hooks — Node knows about the map before your code runs, so nothing needs `outFiles`.

`ts-node` respects your `tsconfig.json`. If `sourceMap` is **false** there, `ts-node` emits no maps and breakpoints will not bind — even though nothing is "compiled to disk". This surprises people constantly:

```jsonc
// ❌ ts-node with no source maps — breakpoints never bind
{ "compilerOptions": { "sourceMap": false } }

// ✅ ts-node needs this even though nothing is written to disk
{ "compilerOptions": { "sourceMap": true } }

// Or force it under the ts-node key without changing the build:
{
  "compilerOptions": { "sourceMap": false },
  "ts-node": { "compilerOptions": { "sourceMap": true }, "transpileOnly": true }
}
```

### Jest, transforms, and the stale-cache trap

Jest transforms every file through `babel-jest`, `ts-jest`, or `@swc/jest`, and **caches the result** in `/tmp/jest_*`. The cache key includes file contents and config — but not always your `tsconfig` changes. Symptom: you enable `sourceMap`, restart, and breakpoints still don't bind.

```bash
npx jest --clearCache        # nuke it, then re-run
```

Also make sure the transformer is told to emit maps:

```js
// jest.config.js
export default {
  preset: "ts-jest",
  transform: {
    "^.+\\.tsx?$": ["ts-jest", {
      tsconfig: "tsconfig.test.json",
      sourceMap: true,          // ts-jest honours this explicitly
      isolatedModules: true     // faster; skips cross-file type checks
    }]
  },
  testEnvironment: "node",
  // Coverage instrumentation rewrites code and can misalign maps — turn it off while debugging:
  collectCoverage: false
};
```

And the two flags that matter most: `--runInBand` (breakpoints can't reach worker child processes) and `--testTimeout` (a 5s default kills your session while you're reading a variable).

### `restart: true` and the nodemon/tsx-watch loop

When a watcher restarts the process, the old inspector socket dies and the debug session ends. `"restart": true` on an **attach** config makes VS Code poll and reconnect.

```jsonc
{
  "type": "node", "request": "attach", "name": "Attach (watch mode)",
  "port": 9229,
  "restart": { "delay": 500, "maxAttempts": 20 },
  "outFiles": ["${workspaceFolder}/dist/**/*.js"]
}
```

Paired with:

```json
{ "scripts": { "dev": "tsx watch --inspect src/server.ts" } }
```

Breakpoints survive across restarts because VS Code re-sends them on each reconnect.

### The performance cost of source maps

Not free, but usually irrelevant:

- **Build time:** `sourceMap: true` adds roughly 5–15% to a `tsc` build.
- **Startup:** `--enable-source-maps` parses maps lazily — cost is paid on the first thrown error, typically 10–50ms for a mid-size service.
- **Memory:** parsed maps for a large service can hold tens of MB. Real, but far cheaper than an unreadable stack trace at 3am.
- **`inlineSourceMap` + `inlineSources`** can triple the size of `dist/`. On Lambda, that eats into the deployment package limit and cold start.

Ship source maps to production. Ship them privately (don't serve `.map` over HTTP from a public path), but ship them.

### Debugging TypeScript that never hits disk

With `tsx`, `ts-node`, `vite-node`, or a bundler in watch mode, there is no `dist/`. `outFiles` is then meaningless — remove it, or VS Code wastes startup time scanning an empty glob. The maps are inline, delivered through Node's loader hooks. If breakpoints don't bind in this setup, the cause is almost always one of:

1. The loader isn't actually active (`runtimeExecutable` pointed at `node` instead of `tsx`).
2. Something in the chain stripped the `sourceMappingURL` comment.
3. You're debugging a *child* process the launch config never attached to (`--runInBand` again, or `child_process.fork` without `--inspect` inheritance).

---

## Common mistakes

### Mistake 1 — Setting `program` to the TypeScript file and expecting Node to run it

```jsonc
// ❌ Node cannot execute .ts. You get:
//    SyntaxError: Unexpected token ':'   (or ERR_UNKNOWN_FILE_EXTENSION on ESM)
{
  "type": "node",
  "request": "launch",
  "name": "Debug",
  "program": "${workspaceFolder}/src/server.ts"
}
```

```jsonc
// ✅ Option A — run the compiled output, and build first:
{
  "type": "node",
  "request": "launch",
  "name": "Debug (compiled)",
  "program": "${workspaceFolder}/dist/server.js",
  "preLaunchTask": "npm: build",
  "outFiles": ["${workspaceFolder}/dist/**/*.js"]
}

// ✅ Option B — use a runtime that understands TypeScript:
{
  "type": "node",
  "request": "launch",
  "name": "Debug (tsx)",
  "runtimeExecutable": "tsx",
  "args": ["${workspaceFolder}/src/server.ts"]
}
```

### Mistake 2 — Forgetting `outFiles`, then blaming source maps

```jsonc
// ❌ sourceMap is on, dist/*.js.map exist, and breakpoints STILL don't bind —
//    because VS Code was never told where to look for them before startup.
{
  "type": "node",
  "request": "launch",
  "program": "${workspaceFolder}/dist/server.js"
}
```

```jsonc
// ✅ Point at the output tree, recursively:
{
  "type": "node",
  "request": "launch",
  "program": "${workspaceFolder}/dist/server.js",
  "outFiles": ["${workspaceFolder}/dist/**/*.js"]
}
```

Quick check before you touch config: does `dist/server.js.map` actually exist, and does `dist/server.js` end with `//# sourceMappingURL=server.js.map`? If not, the problem is `tsconfig`, not `launch.json`.

### Mistake 3 — Debugging against a stale `dist/`

```jsonc
// ❌ No preLaunchTask. You edit src/, hit F5, and Node runs yesterday's build.
//    Symptom: the debugger stops on a line that doesn't match what you see,
//    or your new code "isn't running", or a fixed bug still reproduces.
{ "type": "node", "request": "launch", "program": "${workspaceFolder}/dist/server.js" }
```

```jsonc
// ✅ Always build first — and clean, so deleted files don't linger in dist/:
{
  "type": "node",
  "request": "launch",
  "program": "${workspaceFolder}/dist/server.js",
  "preLaunchTask": "npm: build",
  "outFiles": ["${workspaceFolder}/dist/**/*.js"]
}
// package.json: { "scripts": { "build": "rimraf dist && tsc -p tsconfig.json" } }
```

### Mistake 4 — Running Jest in parallel and wondering where the breakpoints went

```jsonc
// ❌ Jest forks worker processes. The debugger is attached to the PARENT only.
//    Breakpoints inside tests never fire; the run just completes.
{
  "type": "node", "request": "launch",
  "program": "${workspaceFolder}/node_modules/jest/bin/jest.js",
  "args": ["${relativeFile}"]
}
```

```jsonc
// ✅ Force a single process, disable the transform cache, and raise the timeout:
{
  "type": "node", "request": "launch",
  "program": "${workspaceFolder}/node_modules/jest/bin/jest.js",
  "args": ["--runInBand", "--no-cache", "--testTimeout=600000", "${relativeFile}"],
  "console": "integratedTerminal"
}
```

### Mistake 5 — Stack traces in production still pointing at `dist/`

```ts
// ❌ Source maps were built but nothing tells the RUNTIME to use them.
//    Building maps affects the debugger; it does not affect Error.stack.
//    Dockerfile: CMD ["node", "dist/server.js"]
//    Log output: at createOrder (/app/dist/services/orderService.js:214:19)
```

```ts
// ✅ Enable them at runtime — pick one:

// (a) Node's built-in flag — preferred, no dependency:
//     CMD ["node", "--enable-source-maps", "dist/server.js"]
//     or  ENV NODE_OPTIONS="--enable-source-maps"

// (b) The package, imported as the very first line of your entry file:
import "source-map-support/register.js";

// And make sure the maps actually SHIP. This Dockerfile line silently breaks both:
//   COPY --from=build /app/dist/*.js ./dist/     ← .map files excluded! ❌
//   COPY --from=build /app/dist ./dist           ← maps included        ✅
```

### Mistake 6 — Using `sourceRoot` to "fix" paths on your own machine

```jsonc
// ❌ Someone's breakpoints bound to the wrong file, so they added:
{ "compilerOptions": { "sourceMap": true, "sourceRoot": "/Users/alice/repos/api/src" } }
// Now the build only debugs correctly on Alice's laptop, and CI artifacts are poisoned.
```

```jsonc
// ✅ Leave sourceRoot alone. Fix it debug-side, per environment, in launch.json:
{
  "type": "node", "request": "attach", "port": 9229,
  "localRoot": "${workspaceFolder}",
  "remoteRoot": "/app"
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a scratch project and get a breakpoint to bind against compiled output — no shortcuts, no `tsx`.

Concretely:

1. `mkdir ts-debug-lab && cd ts-debug-lab && npm init -y && npm i -D typescript @types/node`
2. Write a `tsconfig.json` with `rootDir: "src"`, `outDir: "dist"`, `sourceMap: true`, `strict: true`, `target: "ES2022"`, `module: "NodeNext"`.
3. Write `src/index.ts` containing a function `calculateOrderTotal(items: { sku: string; priceCents: number; quantity: number }[]): number` and call it with three items.
4. Build, then confirm from the terminal that `dist/index.js.map` exists **and** that `dist/index.js` ends with a `sourceMappingURL` comment.
5. Write `.vscode/tasks.json` with an `npm: build` task, and `.vscode/launch.json` with a launch config that has `program`, `preLaunchTask`, `outFiles`, and `skipFiles`.
6. Set a breakpoint inside the reduce/loop in `calculateOrderTotal`, press F5, and confirm the dot goes solid red and the Variables pane shows the current item.
7. Now **deliberately break it**: set `sourceMap: false`, rebuild, relaunch. Record exactly what changes in the gutter and in the Call Stack pane. Set it back.

```
// Write your code here
```

### Exercise 2 — medium

Build a `launch.json` for a real service with four working configurations, and prove each one.

Concretely:

1. Scaffold a small Express (or Fastify) API in TypeScript with at least: `src/server.ts`, `src/routes/userRoutes.ts`, `src/services/userService.ts`. Include one route that throws deliberately (`GET /users/:userId/boom`).
2. Add these four configurations and verify each by binding a breakpoint inside `userService.ts`:
   - **Launch compiled** — with `preLaunchTask`, `outFiles`, `envFile`.
   - **Launch via tsx** — using `runtimeExecutable`, no build step, no `outFiles`.
   - **Attach to :9229** — you start `node --inspect dist/server.js` in a terminal yourself, then attach. Add `"restart": true` and confirm it survives a manual restart.
   - **Attach with `--inspect-brk`** — prove that a breakpoint on a module top-level statement (e.g. config validation in `src/config.ts`) fires, and explain in a comment why plain `--inspect` misses it.
3. Add a `compounds` entry that launches the API and a second `src/worker.ts` process together with `stopAll: true`.
4. Add a conditional breakpoint that only fires for `userId === 4711`, and a **logpoint** that prints `handling {userId} for {req.headers['x-request-id']}` without stopping.
5. Wire up `--enable-source-maps` and hit `/users/1/boom`. Confirm the stack trace names `.ts` files and correct line numbers. Then remove the flag and record the difference.
6. Add `skipFiles` entries until stepping into the route handler no longer takes you through Express internals.

```
// Write your code here
```

### Exercise 3 — hard

Reproduce and fix every classic "breakpoints won't bind" failure, then produce a diagnostic checklist from what you learn.

Concretely:

1. **Docker attach.** Write a two-stage `Dockerfile` for the Exercise 2 service. The final stage must copy `dist/` *including* `.map` files. Run with `--inspect=0.0.0.0:9229`, publish 9229 in `docker-compose.yml`, and write an attach config with `localRoot`/`remoteRoot` that binds a breakpoint inside the container. Then break it on purpose by changing the `COPY` to exclude `.map` files, and document the exact symptom.
2. **Path remapping.** Build the image with `"sourceRoot": "/app/src"` and observe how the debugger behaves *without* `localRoot`/`remoteRoot`. Then remove `sourceRoot` and fix it purely from `launch.json`. Write two sentences on which approach you'd standardise on and why.
3. **Test debugging.** Add both Jest and Vitest to the project with at least one meaningful test of `userService`. Produce a debug config for each that binds a breakpoint *inside the test* and *inside the code under test*. For Jest, demonstrate the stale-cache failure and fix it with `--clearCache`. For Vitest, demonstrate that parallel pools break breakpoints and fix it with single-fork options.
4. **Monorepo output roots.** Split the project into `packages/shared` (built to its own `dist/`) and `services/api`. Make a single `outFiles` array that covers both, and confirm you can Step Into a function defined in `packages/shared` from a breakpoint in `services/api` and land on the **`.ts`** source, not the `.d.ts` or the `.js`.
5. **Source-mapped error reporting.** Write `src/bootstrap/sourceMaps.ts` that installs `source-map-support`, sets `Error.stackTraceLimit`, and registers `uncaughtException`/`unhandledRejection` handlers that log structured JSON with a mapped stack. Prove the mapping works from a `node dist/server.js` run with no debugger attached at all.
6. Finally, write `DEBUGGING.md` in the repo: a symptom → cause → fix table covering at least eight distinct failure modes you actually hit while doing steps 1–5.

```
// Write your code here
```

---

## Quick reference cheat sheet

```jsonc
// tsconfig.json
"sourceMap": true          // separate .js.map — the default choice
"inlineSourceMap": true    // map embedded in .js — mutually exclusive with sourceMap
"inlineSources": true      // original .ts text inside the map — works with either
"sourceRoot": "/app/src"   // path prefix for map "sources" — only for build≠debug paths
"declarationMap": true     // .d.ts → .ts mapping, for go-to-definition
```

```jsonc
// launch.json essentials
"program"           // JS entry point Node runs (never a .ts file)
"runtimeExecutable" // use "tsx" / "ts-node" instead of plain node
"runtimeArgs"       // e.g. ["-r","ts-node/register"] or ["--enable-source-maps"]
"outFiles"          // globs of emitted JS to scan for maps — fixes unbound breakpoints
"skipFiles"         // ["<node_internals>/**","**/node_modules/**"] — clean stepping
"smartStep"         // skip unmapped generated code (__awaiter, __decorate)
"preLaunchTask"     // "npm: build" — never debug a stale dist/
"envFile" / "env"   // env vars; "env" overrides "envFile"
"console"           // "integratedTerminal" for a real TTY
"localRoot"/"remoteRoot" // host ↔ container path remapping for attach
"restart"           // reattach after nodemon/tsx-watch restarts
"port" / "address"  // attach target — 9229 by default
```

```bash
node --inspect app.js            # inspector on, run immediately
node --inspect-brk app.js        # inspector on, pause at line 1
node --inspect=0.0.0.0:9229 app  # bind all interfaces (dev containers only)
node --enable-source-maps app.js # map Error.stack back to .ts at runtime
npx jest --runInBand --no-cache  # single process, fresh transforms
npx jest --clearCache            # nuke stale transform cache
```

| Symptom | Most likely cause | Fix |
|---|---|---|
| Hollow grey breakpoint | No map found | `sourceMap: true` + correct `outFiles` |
| Stops on the wrong line | Stale `dist/` | Add `preLaunchTask: "npm: build"` |
| Stops in `.js` not `.ts` | `sources` don't resolve | Check `rootDir` / `localRoot`+`remoteRoot` |
| Misses startup code | Attached too late | `--inspect-brk` instead of `--inspect` |
| Jest breakpoints ignored | Worker child processes | `--runInBand` |
| Vitest breakpoints ignored | Thread/fork pool | `--pool=forks --poolOptions.forks.singleFork` |
| Step Into lands in Express | No `skipFiles` | Add `**/node_modules/**` |
| Prod stack shows `dist/*.js` | Runtime maps off | `--enable-source-maps` or `source-map-support` |
| Works locally, not in Docker | Path mismatch / maps not copied | `localRoot`+`remoteRoot`; `COPY dist ./dist` |
| ts-node won't bind | `sourceMap: false` in tsconfig | Enable it, or set it under the `ts-node` key |

---

## Connected topics

- **03 — tsconfig in depth** — `sourceMap`, `inlineSources`, `outDir`, and `rootDir` all live here; debugging config is tsconfig config.
- **04 — TypeScript Node.js project setup** — the `tsx` / `ts-node` dev loop and npm scripts that the launch configurations above hook into.
- **76 — Building for production** — why `--enable-source-maps` and multi-stage Docker `COPY` decisions determine whether your production stack traces are readable.
- **77 — Monorepo basics with TypeScript** — multi-root `outFiles`, and how `declarationMap` makes Step Into land on `.ts` source across package boundaries.
