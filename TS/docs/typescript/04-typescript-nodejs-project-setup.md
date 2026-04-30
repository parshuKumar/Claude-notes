# 04 — TypeScript with Node.js Project Setup

## What is this?

This topic is the full, production-ready project scaffold for a Node.js + TypeScript backend — from an empty folder to a running Express server with the correct folder structure, `package.json` scripts, `tsconfig.json`, type definitions, and development workflow. Think of it as the "create-react-app" equivalent but done manually so you understand every single piece.

## Why does it matter?

Every TypeScript backend project you work on professionally will follow a structure like this. Getting it wrong leads to: compiled files mixed with source files, broken production builds, imports that work in dev but crash in prod, missing type definitions, and `npm start` doing nothing useful. Do this once correctly and you'll replicate it forever.

## The JavaScript way vs the TypeScript way

### JavaScript Node.js project — minimal setup

```
my-api/
├── node_modules/
├── src/
│   └── index.js        ← just write JS, node runs it directly
├── package.json
└── .gitignore
```

```json
// package.json scripts
{
  "scripts": {
    "dev": "nodemon src/index.js",
    "start": "node src/index.js"
  }
}
```

No compiler. No config. `nodemon src/index.js` and you're live.

### TypeScript Node.js project — what changes

```
my-api/
├── src/                ← ALL TypeScript source files
│   └── index.ts        ← entry point (.ts not .js)
├── dist/               ← compiled JS output (auto-generated, never edit)
├── node_modules/
├── package.json
├── tsconfig.json       ← compiler configuration
└── .gitignore
```

```json
// package.json scripts
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

Three scripts instead of two: `dev` for development (ts-node, no compile step), `build` to compile, `start` to run compiled output.

---

## Full setup walkthrough — step by step

### Step 1 — Initialize the project

```bash
mkdir my-api
cd my-api
npm init -y
```

This creates `package.json`. The `-y` flag accepts all defaults.

---

### Step 2 — Install all dependencies

```bash
# TypeScript compiler and dev tools
npm install --save-dev typescript ts-node nodemon

# Type definitions for Node.js built-ins (fs, path, process, http, etc.)
npm install --save-dev @types/node

# Express (runtime dependency — goes to production)
npm install express

# Type definitions for Express (dev only — stripped at compile time)
npm install --save-dev @types/express
```

**Why `--save-dev` for TypeScript packages?**

TypeScript, ts-node, nodemon, and all `@types/*` packages are **development tools only**. They don't run in production. Your production server runs compiled JavaScript — it never touches TypeScript. Keep `devDependencies` lean.

```json
// What your package.json dependencies look like:
{
  "dependencies": {
    "express": "^4.18.0"        // runs in production
  },
  "devDependencies": {
    "typescript": "^5.4.0",     // dev only
    "ts-node": "^10.9.0",       // dev only
    "nodemon": "^3.1.0",        // dev only
    "@types/node": "^20.0.0",   // dev only
    "@types/express": "^4.17.21" // dev only
  }
}
```

---

### Step 3 — Create tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "sourceMap": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

### Step 4 — Create the folder structure

```bash
mkdir src
mkdir src/routes
mkdir src/controllers
mkdir src/middleware
mkdir src/types
mkdir src/utils
mkdir src/config
```

Full structure:

```
my-api/
├── src/
│   ├── index.ts              ← app entry point: creates server, connects DB
│   ├── app.ts                ← Express app setup: middleware, routes (no listen)
│   ├── routes/
│   │   └── userRoutes.ts     ← route definitions
│   ├── controllers/
│   │   └── userController.ts ← request/response handler logic
│   ├── middleware/
│   │   └── errorHandler.ts   ← global error handling middleware
│   ├── types/
│   │   └── index.ts          ← shared TypeScript types for this project
│   ├── utils/
│   │   └── logger.ts         ← utility helpers
│   └── config/
│       └── index.ts          ← environment variables, app config
├── dist/                     ← compiled output (git-ignored)
├── node_modules/
├── .gitignore
├── package.json
└── tsconfig.json
```

**Why split `app.ts` and `index.ts`?**

`app.ts` sets up Express (middleware, routes) but does NOT call `app.listen()`. `index.ts` imports the app and starts the server. This separation makes testing easy — you can import `app` in tests without binding to a port.

---

### Step 5 — Write package.json scripts

```json
{
  "name": "my-api",
  "version": "1.0.0",
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "type-check": "tsc --noEmit"
  }
}
```

| Script | Command | What it does |
|--------|---------|-------------|
| `npm run dev` | `nodemon --exec ts-node src/index.ts` | Hot-reload dev server using ts-node |
| `npm run build` | `tsc` | Compiles all `.ts` in `src/` to `dist/` |
| `npm start` | `node dist/index.js` | Runs the compiled production build |
| `npm run type-check` | `tsc --noEmit` | Type-checks WITHOUT outputting files — useful in CI |

---

### Step 6 — Create .gitignore

```
node_modules/
dist/
.env
*.js.map
```

`dist/` is generated — never commit it. `node_modules/` is obvious. `.env` contains secrets. `*.js.map` source maps are usually large and only needed in specific setups.

---

## Example 1 — basic: the minimal working server

**`src/config/index.ts`**

```ts
// Centralize all environment variables here — typed
const config = {
  port: Number(process.env.PORT) || 3000,       // convert string env var to number
  nodeEnv: process.env.NODE_ENV || 'development',
};

export default config;
```

**`src/types/index.ts`**

```ts
// Shared types used across the app
export type ApiResponse<T> = {
  success: boolean;   // was the request successful?
  data: T | null;     // the payload (null on error)
  error: string | null; // error message (null on success)
};
```

**`src/app.ts`**

```ts
import express, { Application } from 'express';  // Application is the Express app type
import userRoutes from './routes/userRoutes';

const app: Application = express();

// Middleware
app.use(express.json());                         // parse JSON request bodies
app.use(express.urlencoded({ extended: true })); // parse URL-encoded bodies

// Routes
app.use('/api/users', userRoutes);

// Health check
app.get('/health', (_req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

export default app;
```

**`src/index.ts`**

```ts
import app from './app';         // import the configured Express app
import config from './config';   // import typed config

app.listen(config.port, () => {
  console.log(`Server running on port ${config.port} in ${config.nodeEnv} mode`);
});
```

**`src/routes/userRoutes.ts`**

```ts
import { Router } from 'express';
import { getAllUsers, getUserById } from '../controllers/userController';

const router = Router();

router.get('/', getAllUsers);           // GET /api/users
router.get('/:userId', getUserById);   // GET /api/users/:userId

export default router;
```

**`src/controllers/userController.ts`**

```ts
import { Request, Response } from 'express';
import { ApiResponse } from '../types';

// Simulate a user type
type User = {
  id: number;
  name: string;
  email: string;
};

// Fake data store
const users: User[] = [
  { id: 1, name: 'Parsh', email: 'parsh@dev.io' },
  { id: 2, name: 'Alice', email: 'alice@dev.io' },
];

export function getAllUsers(_req: Request, res: Response): void {
  const response: ApiResponse<User[]> = {
    success: true,
    data: users,
    error: null,
  };
  res.json(response);
}

export function getUserById(req: Request, res: Response): void {
  const userId = parseInt(req.params.userId, 10);  // always parse to number

  if (isNaN(userId)) {
    const response: ApiResponse<null> = {
      success: false,
      data: null,
      error: 'Invalid user ID — must be a number',
    };
    res.status(400).json(response);
    return;
  }

  const user = users.find((u) => u.id === userId);

  if (!user) {
    const response: ApiResponse<null> = {
      success: false,
      data: null,
      error: `User with ID ${userId} not found`,
    };
    res.status(404).json(response);
    return;
  }

  const response: ApiResponse<User> = {
    success: true,
    data: user,
    error: null,
  };
  res.json(response);
}
```

**Run it:**

```bash
npm run dev
# nodemon starts, ts-node compiles in memory
# Server running on port 3000 in development mode

# Test it:
# GET http://localhost:3000/health
# GET http://localhost:3000/api/users
# GET http://localhost:3000/api/users/1
# GET http://localhost:3000/api/users/99  ← returns 404
```

---

## Example 2 — real world backend use case

### Adding a typed error handler middleware

**`src/middleware/errorHandler.ts`**

```ts
import { Request, Response, NextFunction } from 'express';

// Custom error class — extends the built-in Error with a status code
export class AppError extends Error {
  statusCode: number;
  isOperational: boolean;   // operational = expected error (404, 400), not a bug

  constructor(message: string, statusCode: number) {
    super(message);              // call Error constructor
    this.statusCode = statusCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);  // clean stack trace
  }
}

// Global error handler — must have exactly 4 parameters for Express to recognize it
export function errorHandler(
  err: AppError | Error,        // can be our AppError or a generic Error
  _req: Request,
  res: Response,
  _next: NextFunction           // must be declared even if unused
): void {
  // Check if it's our known AppError
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      success: false,
      data: null,
      error: err.message,
    });
    return;
  }

  // Unknown error — don't leak internals to the client
  console.error('Unexpected error:', err);
  res.status(500).json({
    success: false,
    data: null,
    error: 'An unexpected error occurred',
  });
}
```

**Register it in `src/app.ts`** (must be last middleware):

```ts
import express, { Application } from 'express';
import userRoutes from './routes/userRoutes';
import { errorHandler } from './middleware/errorHandler';

const app: Application = express();
app.use(express.json());
app.use('/api/users', userRoutes);

// Error handler MUST be registered after all routes
app.use(errorHandler);

export default app;
```

**Use it in a controller:**

```ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../middleware/errorHandler';

export async function deleteUser(
  req: Request,
  res: Response,
  next: NextFunction     // pass errors to the error handler
): Promise<void> {
  const userId = parseInt(req.params.userId, 10);

  if (isNaN(userId)) {
    // Throw a typed error — the error handler catches it
    next(new AppError('Invalid user ID', 400));
    return;
  }

  // simulate DB operation
  const deleted = true;  // pretend delete happened

  if (!deleted) {
    next(new AppError(`User ${userId} not found`, 404));
    return;
  }

  res.status(204).send();  // 204 No Content on successful delete
}
```

---

## The `nodemon.json` alternative

Instead of a long inline `nodemon` command in `package.json`, put this at the root:

**`nodemon.json`**

```json
{
  "watch": ["src"],
  "ext": "ts,json",
  "ignore": ["src/**/*.spec.ts", "src/**/*.test.ts"],
  "exec": "ts-node src/index.ts"
}
```

Then `package.json` becomes clean:

```json
{
  "scripts": {
    "dev": "nodemon",
    "build": "tsc",
    "start": "node dist/index.js",
    "type-check": "tsc --noEmit"
  }
}
```

---

## Building and running production

```bash
# Step 1: Compile
npm run build

# What tsc produces:
dist/
  index.js
  app.js
  routes/
    userRoutes.js
  controllers/
    userController.js
  middleware/
    errorHandler.js
  types/
    index.js         ← empty — only exported types, no runtime code
  config/
    index.js
  *.js.map           ← source maps (if sourceMap: true)

# Step 2: Run
npm start
# node dist/index.js
```

**Tip:** On a real production server (Heroku, Railway, Render, EC2), you run `npm run build` during the deploy step, then `npm start` to launch the server.

---

## Common mistakes

### Mistake 1 — Putting TypeScript devDependencies in `dependencies`

```json
// ❌ WRONG — shipping TypeScript itself to production
{
  "dependencies": {
    "typescript": "^5.4.0",
    "ts-node": "^10.9.0",
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.21",
    "express": "^4.18.0"
  }
}
// Production servers run npm install --omit=dev
// They only need express. Shipping typescript/ts-node wastes space and time.

// ✅ RIGHT
{
  "dependencies": {
    "express": "^4.18.0"         // runtime dep
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "ts-node": "^10.9.0",
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.21"
  }
}
```

### Mistake 2 — Using `npm start` in development

```bash
# ❌ WRONG — npm start runs compiled JS, not source TS
npm start           # runs: node dist/index.js
# Edits to .ts files do NOTHING — you're running old compiled code

# ✅ RIGHT — use npm run dev for development
npm run dev         # runs: nodemon --exec ts-node src/index.ts
# Edits to .ts files → nodemon restarts → ts-node recompiles → live
```

### Mistake 3 — Not separating `app.ts` from `index.ts`

```ts
// ❌ WRONG — calling app.listen() inside the same file you export the app from
// src/app.ts
const app = express();
app.use(express.json());
app.listen(3000);          // starts server immediately on import
export default app;

// Now in your test file:
import app from '../src/app';   // ← importing this STARTS THE SERVER
// Your tests now spin up a real server for every test file

// ✅ RIGHT — separate concerns
// src/app.ts — just app setup, no listen()
export default app;

// src/index.ts — only file that calls listen()
import app from './app';
app.listen(3000);

// test files import app safely without starting the server
import app from '../src/app';
```

---

## Practice exercises

### Exercise 1 — easy

Create the complete folder structure and files for a new TypeScript Node.js project from scratch. No Express — just Node.js. The project should have:

- `package.json` with all 4 scripts (`dev`, `build`, `start`, `type-check`)
- `tsconfig.json` with strict mode, `src/` → `dist/`
- `src/config/index.ts` that exports a typed config object with `port: number` and `appName: string`
- `src/index.ts` that imports the config and logs `"[appName] starting on port [port]"`

Run `npm run dev` and confirm it works. Then run `npm run build` and inspect `dist/`.

```ts
// Write your src/config/index.ts here
// Write your src/index.ts here
```

### Exercise 2 — medium

Extend the project from Exercise 1 by adding Express. Create the `app.ts` / `index.ts` split pattern properly.

- `src/app.ts` — creates Express app, registers `express.json()`, adds a `GET /health` route returning `{ status: "ok", timestamp: new Date() }`
- `src/index.ts` — imports the app and config, calls `app.listen(config.port, callback)`
- `src/types/index.ts` — defines and exports `ApiResponse<T>` type
- `src/routes/statusRoutes.ts` — a router with one route: `GET /status` that returns `ApiResponse<{ uptime: number; version: string }>` (read version from `package.json` using `resolveJsonModule`)

Mount the status router in `app.ts` at `/api/status`.

```ts
// Write all four files here
```

### Exercise 3 — hard

Build a mini user management API on top of the structure from Exercise 2. Requirements:

1. `src/types/index.ts` — define `User` type (`id`, `name`, `email`, `role: "admin" | "user"`), `CreateUserBody` type (name, email, role), and `ApiResponse<T>`.
2. `src/controllers/userController.ts` — implement `getAllUsers`, `getUserById`, `createUser`. `createUser` reads from `req.body` typed as `CreateUserBody`. If `name` or `email` is missing, return a 400 with an error response. Otherwise add to an in-memory array and return 201 with the new user.
3. `src/routes/userRoutes.ts` — wire up `GET /`, `GET /:userId`, `POST /`.
4. `src/middleware/errorHandler.ts` — create the `AppError` class and `errorHandler` middleware exactly as shown in Example 2.
5. Mount user routes at `/api/users` and register the error handler last.

All functions must have complete TypeScript types. `npm run type-check` must pass with zero errors.

```ts
// Write all files here
```

---

## Quick reference cheat sheet

| File | Purpose |
|------|---------|
| `src/index.ts` | Entry point — only calls `app.listen()` |
| `src/app.ts` | Express setup — middleware and routes, no `listen()` |
| `src/routes/*.ts` | Route definitions using Express `Router` |
| `src/controllers/*.ts` | Request handlers — business logic |
| `src/middleware/*.ts` | Express middleware (auth, errors, logging) |
| `src/types/index.ts` | Shared TypeScript types for the project |
| `src/config/index.ts` | Typed env vars and app configuration |
| `src/utils/*.ts` | Pure helper functions |
| `dist/` | Compiled JS output — never edit, never commit |
| `tsconfig.json` | TypeScript compiler configuration |
| `nodemon.json` | Nodemon watcher configuration |

| Script | When to use |
|--------|------------|
| `npm run dev` | Development — hot reload with ts-node |
| `npm run build` | Before deploying — compile TS → JS |
| `npm start` | Production — run compiled `dist/index.js` |
| `npm run type-check` | CI/pre-commit — check types without compiling |

## Connected topics

- **03 — tsconfig.json in depth** — Every option in the `tsconfig.json` used here explained.
- **52 — Typing Express routes** — Deep dive into `Request`, `Response`, `NextFunction` types.
- **53 — Extending Express Request** — Adding `req.user` and typed body to Express.
- **55 — Typing environment variables** — Making `process.env` fully typed and safe.
