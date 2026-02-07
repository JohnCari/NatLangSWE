# The Power of 10 Rules for Safety-Critical Bun (TypeScript Toolchain) Configuration

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

**Note:** Bun is an all-in-one TypeScript runtime, package manager, bundler, and test runner that replaces Node.js, npm/yarn/pnpm, webpack/esbuild, and Jest/Vitest. These rules treat `package.json` as the single source of truth, `bun.lock` as the integrity contract, and assume all code is written in TypeScript (`.ts` / `.tsx`). For TypeScript language-level rules, see the [TypeScript Power of 10](../../languages/typescript/POWER_OF_10.md). For React-specific patterns, see the [React Power of 10](../../frameworks/react/POWER_OF_10.md).

---

## The Original 10 Rules (C Language)

| # | Rule |
|---|------|
| 1 | Restrict all code to very simple control flow constructs—no `goto`, `setjmp`, `longjmp`, or recursion |
| 2 | Give all loops a fixed upper bound provable by static analysis |
| 3 | Do not use dynamic memory allocation after initialization |
| 4 | No function longer than one printed page (~60 lines) |
| 5 | Assertion density: minimum 2 assertions per function |
| 6 | Declare all data objects at the smallest possible scope |
| 7 | Check all return values and validate all function parameters |
| 8 | Limit preprocessor to header includes and simple macros |
| 9 | Restrict pointers: single dereference only, no function pointers |
| 10 | Compile with all warnings enabled; use static analyzers daily |

---

## The Power of 10 Rules — Bun Edition

### Rule 1: Simple Control Flow — Single Toolchain, No Multi-Tool Chains

**Original Intent:** Eliminate complex control flow that impedes static analysis and can cause stack overflows.

**Bun Adaptation:**

```bash
# BAD: Multi-tool chain with mixed runtimes
nvm use 18
npm install
npx webpack --config webpack.config.js
node -r ts-node/register src/index.ts
npx jest --config jest.config.js

# BAD: Mixing package managers
npm install express
yarn add lodash
pnpm add zod
# package-lock.json, yarn.lock, and pnpm-lock.yaml all exist — chaos

# BAD: Shell scripts to glue tools together
#!/bin/bash
npm run build
node scripts/post-build.js
npx copyfiles -u 1 "src/**/*.json" dist/
node dist/index.js

# GOOD: Bun handles everything — one tool, one lockfile
bun install
bun run src/index.ts       # Native TypeScript execution
bun test                    # Built-in test runner
bun build ./src/index.ts --outdir ./dist

# GOOD: Simple package.json scripts
```

```json
{
  "scripts": {
    "dev": "bun run --watch src/index.ts",
    "build": "bun build ./src/index.ts --outdir ./dist --target node",
    "test": "bun test",
    "lint": "biome check src/",
    "typecheck": "tsc --noEmit"
  }
}
```

```bash
# GOOD: Run scripts directly
bun run dev
bun run build
bun run test
```

**Guidelines:**
- Use Bun as the single runtime, package manager, bundler, and test runner
- No mixing package managers — one project, one lockfile (`bun.lock`)
- Use `bun run` for all script execution — no bare `node` or `npx`
- Keep `package.json` scripts simple: one command per script, no `&&` chains
- Delete `package-lock.json`, `yarn.lock`, and `pnpm-lock.yaml` if migrating to Bun

---

### Rule 2: Fixed Bounds — Pin Versions, Lock Everything

**Original Intent:** Ensure all loops terminate and can be analyzed statically.

**Bun Adaptation:**

```json
// BAD: No version constraints
{
  "dependencies": {
    "express": "*",              // Any version — breaking changes guaranteed
    "lodash": "latest",          // Resolves differently every install
    "zod": ">=3.0.0"            // No upper bound
  }
}
// bun.lock not committed to git

// BAD: Inconsistent version strategies
{
  "dependencies": {
    "react": "^18.2.0",         // Caret — allows minor bumps
    "react-dom": "~18.2.0",    // Tilde — allows patch bumps only
    "next": "14.1.0"            // Exact — no bumps at all
    // Pick ONE strategy and stick with it
  }
}

// GOOD: Bounded version specifiers with consistent strategy
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "next": "^14.1.0",
    "zod": "^3.22.0"
  },
  "engines": {
    "bun": ">=1.1.0"
  }
}
// bun.lock committed and verified in CI
```

```bash
# GOOD: Commit the lockfile
git add bun.lock

# GOOD: Verify lockfile is in sync
bun install --frozen-lockfile  # Fails if lockfile is stale
```

**Guidelines:**
- Always commit `bun.lock` to version control
- Use a consistent version strategy across all dependencies (prefer `^` for semver)
- Never use `*` or `latest` as version specifiers
- Specify `"engines"` in `package.json` to document the required Bun version
- Run `bun install --frozen-lockfile` in CI to verify lockfile integrity
- Pin exact versions only when a specific version is required for compatibility

---

### Rule 3: No Dynamic Allocation After Init — Deterministic Installs, No Runtime Package Installation

**Original Intent:** Prevent unbounded memory growth and allocation failures.

**Bun Adaptation:**

```typescript
// BAD: Installing packages at runtime
import { execSync } from "child_process";

function ensureDependency(pkg: string): void {
  try {
    require.resolve(pkg);
  } catch {
    execSync(`bun add ${pkg}`);  // Modifying node_modules at runtime!
  }
}

// BAD: Dynamic require of user-provided module names
function loadPlugin(name: string): unknown {
  return require(name);  // Arbitrary code execution risk
}
```

```dockerfile
# BAD: Dockerfile without lockfile
FROM oven/bun:1
WORKDIR /app
COPY package.json .
RUN bun install
COPY . .
CMD ["bun", "run", "src/index.ts"]

# GOOD: Deterministic Docker build
FROM oven/bun:1 AS base
WORKDIR /app

# Install dependencies first (cached layer)
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production

# Copy source and build
COPY . .
RUN bun build ./src/index.ts --outdir ./dist --target bun

# Minimal production image
FROM oven/bun:1-slim
WORKDIR /app
COPY --from=base /app/dist ./dist
COPY --from=base /app/node_modules ./node_modules
CMD ["bun", "run", "dist/index.js"]
```

```typescript
// GOOD: All dependencies declared in package.json, resolved at install time
import { serve } from "bun";
import { z } from "zod";
import { Hono } from "hono";

// Everything is available — no runtime installs needed
const app = new Hono();
```

**Guidelines:**
- The environment is fully resolved at install time (`bun install`) — never at runtime
- Use `bun install --frozen-lockfile` in CI and Docker (fails if lockfile is stale)
- Never call `bun add`, `npm install`, or `exec("bun install")` from application code
- Docker: install deps first for layer caching, copy source second
- Use multi-stage Docker builds for minimal production images
- All imports must reference packages declared in `package.json`

---

### Rule 4: Keep It Small — Lean package.json, Organized Workspaces

**Original Intent:** Ensure functions are small enough to understand, test, and verify.

**Bun Adaptation:**

```json
// BAD: Monolithic package.json with everything mixed together
{
  "dependencies": {
    "react": "^18.2.0",
    "next": "^14.1.0",
    "express": "^4.18.0",
    "prisma": "^5.10.0",
    "@testing-library/react": "^14.0.0",
    "vitest": "^1.3.0",
    "eslint": "^8.57.0",
    "prettier": "^3.2.0",
    "typescript": "^5.4.0",
    "tailwindcss": "^3.4.0",
    "storybook": "^8.0.0",
    "playwright": "^1.42.0",
    "husky": "^9.0.0",
    "lint-staged": "^15.2.0"
    // ... 30 more mixed deps — which are production? which are dev?
  }
}

// BAD: Deeply nested script chains
{
  "scripts": {
    "dev": "concurrently \"bun run watch:ts\" \"bun run watch:css\" \"bun run watch:server\" \"bun run watch:storybook\"",
    "build": "bun run clean && bun run build:ts && bun run build:css && bun run build:server && bun run copy:assets && bun run generate:sitemap"
  }
}
```

```json
// GOOD: Clean separation of production and dev dependencies
{
  "dependencies": {
    "hono": "^4.1.0",
    "zod": "^3.22.0",
    "drizzle-orm": "^0.30.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/bun": "latest",
    "biome": "^1.6.0",
    "drizzle-kit": "^0.20.0"
  },
  "scripts": {
    "dev": "bun run --watch src/index.ts",
    "build": "bun build ./src/index.ts --outdir ./dist --target bun",
    "test": "bun test",
    "lint": "biome check src/",
    "typecheck": "tsc --noEmit"
  }
}
```

```json
// GOOD: Bun workspaces for monorepos
// root package.json
{
  "name": "myapp",
  "workspaces": ["packages/*", "apps/*"]
}

// apps/web/package.json
{
  "name": "@myapp/web",
  "dependencies": {
    "next": "^14.1.0",
    "react": "^18.2.0",
    "@myapp/ui": "workspace:*"
  }
}

// packages/ui/package.json
{
  "name": "@myapp/ui",
  "dependencies": {
    "react": "^18.2.0"
  }
}
```

**Guidelines:**
- Keep production `dependencies` under 15 entries
- Separate `dependencies` (production) from `devDependencies` (tooling, testing, types)
- Each `scripts` entry should be a single command — no `&&` chains or `concurrently` wrappers
- Use Bun workspaces for monorepos instead of a single bloated `package.json`
- No commented-out dependencies — remove them entirely
- Use `bun add --dev` for dev-only packages

---

### Rule 5: Validate the Environment — Assert with Strict TypeScript and `bun test`

**Original Intent:** Defensive programming catches bugs early; assertions document invariants.

**Bun Adaptation:**

```typescript
// BAD: No runtime validation of external data
async function getUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  const data = await res.json();
  return data as User;  // Type assertion — no actual validation
}

// BAD: Trusting environment variables without validation
const port = process.env.PORT;  // string | undefined — could be anything
Bun.serve({ port });             // What if it's "banana"?

// GOOD: Validate external data at trust boundaries
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(["admin", "user", "guest"]),
});

type User = z.infer<typeof UserSchema>;

async function getUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`Failed to fetch user: ${res.status}`);

  const data = await res.json();
  return UserSchema.parse(data);  // Throws if shape is wrong
}

// GOOD: Validate environment variables at startup
const EnvSchema = z.object({
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  API_KEY: z.string().min(1),
});

const env = EnvSchema.parse(process.env);
// env.PORT is now a validated number, not string | undefined
```

```typescript
// GOOD: Built-in bun test with assertions
import { describe, it, expect } from "bun:test";

describe("UserSchema", () => {
  it("rejects invalid email", () => {
    const result = UserSchema.safeParse({ id: "123", email: "not-an-email", name: "Jo", role: "user" });
    expect(result.success).toBe(false);
  });

  it("accepts valid user", () => {
    const result = UserSchema.safeParse({
      id: "550e8400-e29b-41d4-a716-446655440000",
      email: "user@example.com",
      name: "Jo",
      role: "user",
    });
    expect(result.success).toBe(true);
  });
});
```

```yaml
# GOOD: Full environment validation in CI (GitHub Actions)
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Bun
        uses: oven-sh/setup-bun@v2

      - name: Verify lockfile integrity
        run: bun install --frozen-lockfile

      - name: Type check
        run: bun run typecheck

      - name: Lint
        run: bun run lint

      - name: Run tests
        run: bun test
```

**Guidelines:**
- Validate ALL external data at trust boundaries (API responses, env vars, URL params, file I/O)
- Use Zod or Valibot for runtime schema validation — infer TypeScript types from schemas
- Use `bun test` as the built-in test runner — no need for Jest or Vitest
- Type-check in CI with `tsc --noEmit` — Bun runs TypeScript without type-checking
- Validate environment variables at application startup, not at point of use
- Add `bun install --frozen-lockfile` as the first CI step

---

### Rule 6: Minimal Scope — Scoped Dependencies, No Global Installs

**Original Intent:** Reduce state complexity and potential for misuse.

**Bun Adaptation:**

```bash
# BAD: Global installs polluting the system
bun install -g typescript
bun install -g eslint
bun install -g create-next-app
npm install -g nodemon  # Mixing managers AND installing globally

# BAD: Importing from outside the project
import { utils } from "../../../other-project/src/utils";  // Reaching outside project boundary
import config from "/home/user/global-config.json";         // Absolute path to user home

# GOOD: Everything is a local devDependency or workspace dependency
bun add --dev typescript
bun add --dev @biomejs/biome

# GOOD: Use bunx for one-off commands (no global install needed)
bunx create-next-app@latest my-app
bunx drizzle-kit generate
bunx tsc --noEmit
```

```typescript
// GOOD: Scoped imports from declared dependencies only
import { Hono } from "hono";                    // Declared in package.json
import { z } from "zod";                         // Declared in package.json
import { db } from "@myapp/database";            // Workspace package
import { Button } from "@/components/Button";    // Path alias (tsconfig)

// GOOD: Path aliases instead of deep relative imports
```

```json
// GOOD: tsconfig.json path aliases for clean imports
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"],
      "@lib/*": ["./src/lib/*"]
    }
  }
}
```

```json
// GOOD: Workspace isolation in monorepos
{
  "workspaces": ["apps/*", "packages/*"]
}
// Each package declares its own dependencies
// Shared code is an explicit workspace dependency: "@myapp/shared": "workspace:*"
```

**Guidelines:**
- Never install packages globally — use `bunx` for one-off commands
- All dependencies must be declared in the project's `package.json`
- Use `bun add --dev` for tooling, types, and test utilities
- Use tsconfig `paths` for clean imports instead of deep relative paths (`../../..`)
- In monorepos, each workspace declares its own dependencies explicitly
- No importing from outside the project boundary

---

### Rule 7: Check Return Values — Handle Errors in Bun APIs, Never Swallow Failures

**Original Intent:** Never ignore errors; verify inputs at trust boundaries.

**Bun Adaptation:**

```typescript
// BAD: Ignoring fetch failures
async function getData(url: string) {
  const res = await fetch(url);
  return res.json();  // What if res.ok is false?
}

// BAD: Ignoring file operation errors
const content = Bun.file("config.json").text();  // What if file doesn't exist?

// BAD: Swallowing errors in catch blocks
try {
  await riskyOperation();
} catch {
  // Silent failure — bugs hide here
}

// BAD: No error handling in Bun.serve
Bun.serve({
  fetch(req) {
    const data = JSON.parse(req.body);  // Crashes on invalid JSON
    return Response.json(processData(data));
  },
});
```

```typescript
// GOOD: Check response status before parsing
async function getData(url: string): Promise<ApiResponse> {
  const res = await fetch(url);

  if (!res.ok) {
    throw new Error(`HTTP ${res.status}: ${res.statusText}`);
  }

  const data = await res.json();
  return ApiResponseSchema.parse(data);
}

// GOOD: Handle file operations with existence checks
async function readConfig(path: string): Promise<Config> {
  const file = Bun.file(path);

  if (!(await file.exists())) {
    throw new Error(`Config file not found: ${path}`);
  }

  const text = await file.text();
  return ConfigSchema.parse(JSON.parse(text));
}

// GOOD: Structured error handling in Bun.serve
Bun.serve({
  port: env.PORT,

  async fetch(req): Promise<Response> {
    try {
      const url = new URL(req.url);
      const handler = routes.get(url.pathname);

      if (!handler) {
        return new Response("Not Found", { status: 404 });
      }

      return await handler(req);
    } catch (error) {
      console.error("Unhandled error:", error);
      return new Response("Internal Server Error", { status: 500 });
    }
  },

  error(error): Response {
    console.error("Server error:", error);
    return new Response("Internal Server Error", { status: 500 });
  },
});

// GOOD: Type-safe error handling with Result pattern
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function safeParseJson<T>(
  text: string,
  schema: z.ZodSchema<T>,
): Promise<Result<T>> {
  try {
    const parsed = JSON.parse(text);
    const result = schema.safeParse(parsed);

    if (!result.success) {
      return { ok: false, error: new Error(result.error.message) };
    }

    return { ok: true, value: result.data };
  } catch (error) {
    return { ok: false, error: error instanceof Error ? error : new Error(String(error)) };
  }
}
```

**Guidelines:**
- Always check `response.ok` before calling `.json()` on fetch responses
- Use `Bun.file().exists()` before reading files
- Never use empty `catch {}` blocks — log errors or re-throw
- Implement `error` handler in `Bun.serve()` for uncaught server errors
- Use the Result pattern for operations that can fail expected ways
- Propagate errors to the caller — don't swallow them silently

---

### Rule 8: Limit Metaprogramming — No eval, Explicit Configuration, Minimal Plugins

**Original Intent:** Avoid constructs that create unmaintainable, unanalyzable code.

**Bun Adaptation:**

```typescript
// BAD: Dynamic code execution
function runUserCode(code: string): unknown {
  return eval(code);  // Arbitrary code execution
}

// BAD: Dynamic import of user-provided strings
async function loadModule(name: string): Promise<unknown> {
  return import(name);  // User controls what gets imported
}

// BAD: Complex build plugin chains
// bunfig.toml with 10+ plugins
[build]
plugins = [
  "./plugins/transform-css.ts",
  "./plugins/transform-svg.ts",
  "./plugins/transform-yaml.ts",
  "./plugins/custom-resolver.ts",
  "./plugins/macro-expander.ts",
  // ... each adds hidden behavior
]

// BAD: Monkey-patching globals
Object.defineProperty(globalThis, "fetch", {
  value: customFetch,  // Hidden behavior — breaks debugging
});
```

```typescript
// GOOD: Static imports only
import { Hono } from "hono";
import { z } from "zod";
import { db } from "@/lib/database";

// GOOD: Configuration as typed objects — no dynamic loading
interface AppConfig {
  port: number;
  databaseUrl: string;
  corsOrigins: string[];
}

const config: AppConfig = {
  port: env.PORT,
  databaseUrl: env.DATABASE_URL,
  corsOrigins: ["https://myapp.com"],
};

// GOOD: Bun macros for compile-time code (explicit, traceable)
// macros.ts
export function getVersion(): string {
  const pkg = require("./package.json");
  return pkg.version;
}

// usage.ts — evaluated at bundle time, not runtime
import { getVersion } from "./macros" with { type: "macro" };
const version = getVersion();  // Inlined at build time

// GOOD: Minimal, explicit plugin configuration
// build.ts
await Bun.build({
  entrypoints: ["./src/index.ts"],
  outdir: "./dist",
  target: "bun",
  sourcemap: "linked",
  // Only add plugins with a clear, documented purpose
});
```

```toml
# GOOD: Clean bunfig.toml
# bunfig.toml
[install]
peer = false

[install.scopes]
"@mycompany" = { url = "https://npm.mycompany.com/", token = "$NPM_TOKEN" }
```

**Guidelines:**
- Never use `eval()` or `new Function()` with dynamic input
- No dynamic `import()` with user-provided strings — only static imports or known paths
- Keep `bunfig.toml` minimal — only configure what differs from defaults
- Limit build plugins to what is strictly necessary — each adds hidden complexity
- Use Bun macros (`with { type: "macro" }`) for compile-time transforms instead of runtime magic
- No monkey-patching globals (`fetch`, `console`, `process`)
- Configuration should be typed TypeScript objects, not dynamic JSON loading

---

### Rule 9: Type Safety — Strict TypeScript, No `any`, Typed Bun APIs

**Original Intent:** Restrict pointers: single dereference only, no function pointers.

**Bun Adaptation:**

```typescript
// BAD: Using any
function handleRequest(req: any): any {
  return { data: req.body.data };  // No type safety at all
}

// BAD: Type assertions without validation
const data = (await res.json()) as User;  // Trust without verification

// BAD: Untyped event handlers
Bun.serve({
  fetch(req) {
    const body = req.body;  // ReadableStream | null — needs handling
    const params = new URL(req.url).searchParams;
    const id = params.get("id");  // string | null — unchecked
    return Response.json({ id: id.toUpperCase() });  // Crashes if null!
  },
});

// BAD: Ignoring null/undefined
const file = Bun.file("data.json");
const text = await file.text();  // What if file doesn't exist?
const parsed = JSON.parse(text);  // What if text is empty?
```

```typescript
// GOOD: Strict TypeScript configuration
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noPropertyAccessFromIndexSignature": true,
    "types": ["bun-types"]
  }
}
```

```typescript
// GOOD: Fully typed Bun.serve with proper null handling
import { type Server } from "bun";

const RequestSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

type CreateUserRequest = z.infer<typeof RequestSchema>;

const server: Server = Bun.serve({
  port: env.PORT,

  async fetch(req: Request): Promise<Response> {
    const url = new URL(req.url);

    if (req.method === "POST" && url.pathname === "/users") {
      const body = await req.json().catch(() => null);

      if (!body) {
        return Response.json({ error: "Invalid JSON" }, { status: 400 });
      }

      const result = RequestSchema.safeParse(body);

      if (!result.success) {
        return Response.json({ error: result.error.flatten() }, { status: 400 });
      }

      const user = await createUser(result.data);
      return Response.json(user, { status: 201 });
    }

    return new Response("Not Found", { status: 404 });
  },
});

// GOOD: Typed file operations
async function readJsonFile<T>(path: string, schema: z.ZodSchema<T>): Promise<T> {
  const file = Bun.file(path);

  if (!(await file.exists())) {
    throw new Error(`File not found: ${path}`);
  }

  const text = await file.text();
  return schema.parse(JSON.parse(text));
}

// GOOD: Typed route handlers
type RouteHandler = (req: Request) => Promise<Response>;

const routes = new Map<string, RouteHandler>([
  ["/health", async () => Response.json({ status: "ok" })],
  ["/users", handleUsers],
]);

// GOOD: Discriminated unions for domain types
type ApiResult<T> =
  | { status: "success"; data: T }
  | { status: "error"; code: number; message: string };

function respond<T>(result: ApiResult<T>): Response {
  switch (result.status) {
    case "success":
      return Response.json(result.data);
    case "error":
      return Response.json({ error: result.message }, { status: result.code });
  }
}
```

**Guidelines:**
- Enable all strict TypeScript options — Bun runs TS without type-checking, so `tsc --noEmit` in CI is critical
- Install `bun-types` and add to `tsconfig.json` for proper Bun API types
- Never use `any` — use `unknown` and validate with Zod
- Always handle `null` from `searchParams.get()`, `Map.get()`, and optional chaining
- Type all route handlers explicitly: `(req: Request) => Promise<Response>`
- Use generics for reusable utilities (file readers, API clients, validators)
- Use discriminated unions for domain types and API responses

---

### Rule 10: Static Analysis — Biome, Type Checking, Zero Warnings CI

**Original Intent:** Catch issues at development time; use every available tool.

**Bun Adaptation:**

```yaml
# GOOD: Complete CI pipeline (GitHub Actions)
name: Bun Safety Checks
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Bun
        uses: oven-sh/setup-bun@v2

      - name: Verify lockfile
        run: bun install --frozen-lockfile

      - name: Type check
        run: bunx tsc --noEmit

      - name: Lint and format check
        run: bunx biome ci src/

      - name: Run tests
        run: bun test

      - name: Audit dependencies
        run: bunx npm-audit --audit-level=moderate
```

```json
// GOOD: Biome configuration (biome.json)
{
  "$schema": "https://biomejs.dev/schemas/1.6.0/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "suspicious": {
        "noExplicitAny": "error"
      },
      "complexity": {
        "noForEach": "warn",
        "useSimplifiedLogicExpression": "warn"
      },
      "security": {
        "noDangerouslySetInnerHtml": "error"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2
  }
}
```

```bash
# BAD: No CI verification
# "It works on my machine" — no type checking, no linting, no tests

# BAD: Warnings treated as acceptable
bunx tsc --noEmit  # 47 warnings — ignored
bunx biome check src/  # "just warnings, it's fine"

# GOOD: Zero warnings policy
bunx tsc --noEmit                  # Must pass with zero errors
bunx biome ci src/                 # Lint + format check (fails on any issue)
bun test                           # All tests must pass
```

```typescript
// GOOD: Pre-commit validation with Bun
// scripts/pre-commit.ts
import { $ } from "bun";

const results = await Promise.allSettled([
  $`bunx tsc --noEmit`,
  $`bunx biome check src/`,
  $`bun test`,
]);

const failures = results.filter((r) => r.status === "rejected");
if (failures.length > 0) {
  console.error("Pre-commit checks failed:");
  failures.forEach((f) => console.error(f.reason));
  process.exit(1);
}
```

```json
// GOOD: Automated dependency updates (Renovate)
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchManagers": ["bun"],
      "groupName": "bun dependencies"
    }
  ]
}
```

**Guidelines:**
- Use Biome as the all-in-one linter and formatter (replaces ESLint + Prettier)
- `tsc --noEmit` in CI is mandatory — Bun executes TypeScript without type-checking
- Zero warnings policy — treat all warnings as errors in CI
- Use `bun install --frozen-lockfile` as the first CI step
- Run `bun test` for all tests — no need for external test runners
- Automate dependency updates with Renovate or Dependabot
- Enforce all checks as required status checks on PRs
- Audit dependencies for known vulnerabilities

---

## Summary: Bun Adaptation

| # | Original Rule | Bun Guideline |
|---|---------------|---------------|
| 1 | No goto/recursion | Single toolchain, no multi-tool chains, `bun run` for everything |
| 2 | Fixed loop bounds | Pin versions with `^`, commit `bun.lock`, `--frozen-lockfile` in CI |
| 3 | No dynamic allocation | Deterministic installs, no runtime package installation, multi-stage Docker |
| 4 | 60-line functions | Lean `package.json` (under 15 prod deps), workspaces for monorepos |
| 5 | 2+ assertions/function | Zod validation at boundaries, `bun test`, env validation at startup |
| 6 | Minimize scope | No global installs, `bunx` for one-off commands, tsconfig path aliases |
| 7 | Check returns | Check `response.ok`, handle `null`, Result pattern, `error` handler in `Bun.serve` |
| 8 | Limit preprocessor | No `eval`, static imports only, minimal `bunfig.toml`, typed config objects |
| 9 | Restrict pointers | Strict TypeScript, no `any`, `bun-types`, typed route handlers |
| 10 | All warnings enabled | Biome + `tsc --noEmit` + `bun test` in CI, zero warnings, automated updates |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [Bun Documentation](https://bun.sh/docs) — Oven
- [Bun GitHub Repository](https://github.com/oven-sh/bun) — Oven
- [Bun TypeScript Support](https://bun.sh/docs/runtime/typescript) — Oven
- [Biome Linter](https://biomejs.dev/) — Biome
- [Zod Documentation](https://zod.dev/) — Zod
- [TypeScript Strict Mode](https://www.typescriptlang.org/tsconfig#strict) — Microsoft
- [bun-types](https://www.npmjs.com/package/bun-types) — Oven
