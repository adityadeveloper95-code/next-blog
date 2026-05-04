# Phase 0 — Repository and Toolchain Execution Plan

## Status

Not started.

## Goal

Bootstrap `next-blog-server` as a Node.js + TypeScript backend project with the minimum tooling needed to build, run, and test future phases predictably.

This phase does not implement API behavior yet. It creates the project foundation that later phases will use for Express, Prisma, Redis sessions, validation, middleware, and route/controller/service layering.

## Source Inputs

- `../plan.md`: Phase 0 scope and exit criteria.
- `../backend_architecture.md`: backend stack, folder expectations, and layering.
- `../../hld.md`: API contract, environment needs, and dependency choices implied by PostgreSQL, Redis, sessions, and validation.

## Scope

Phase 0 includes:

- Initialize the Node package and TypeScript compiler setup.
- Add core development scripts for local development, production build/start, tests, and future Prisma commands.
- Choose and install the initial test runner stack.
- Add a minimal source entry point so `npm run build` has a real target.
- Add a minimal smoke test so `npm test` verifies the test stack is wired.
- Add `.env.example` documenting the environment variables required by later phases.
- Add basic repository hygiene files if missing, such as `.gitignore`.

Phase 0 explicitly excludes:

- Express app/middleware wiring.
- Prisma schema and database migrations.
- Redis client/session store setup.
- API routes, controllers, services, validation schemas, and business logic.
- Docker Compose or local database orchestration, unless already required by the repository.

## Tooling Decisions

- Package manager: `pnpm`.
- Runtime: Node.js 22, documented with `.nvmrc` and `package.json` `engines`.
- Module format: ESM using `"type": "module"` and Node-compatible TypeScript settings.
- Test runner: Vitest. Supertest will be added later when HTTP integration tests are introduced.
- ORM commands: defer Prisma scripts until the Prisma setup phase. Phase 0 should not expose `db:migrate` or `db:generate` until Prisma is installed.
- Formatting/linting: include ESLint and Prettier in Phase 0 so style checks are available from the start.

## Target Files

Create or update:

- `package.json`
- `pnpm-lock.yaml`
- `tsconfig.json`
- `eslint.config.js`
- `.prettierrc`
- `.prettierignore`
- `.nvmrc`
- `src/index.ts`
- `test/smoke.test.ts` or `src/**/*.test.ts`
- `.env.example`
- `.gitignore`

Do not create the full architecture folder layout yet unless needed for the minimal build. Phase 1 owns the complete `src/config`, `routes`, `controllers`, `services`, `db`, `middlewares`, `utils`, and `types` structure.

## Dependency Plan

Install development dependencies:

- `typescript`
- `tsx`
- `vitest`
- `@types/node`
- `eslint`
- `prettier`
- `typescript-eslint`
- `@eslint/js`

Install runtime dependencies only if the minimal entry point needs them. Prefer deferring Express, Prisma, Redis, Zod, security middleware, logging, and session packages to Phase 1 when they are wired into the app.

## Script Plan

`package.json` should expose at least:

- `dev`: run the TypeScript entry point locally with watch-friendly tooling.
- `build`: compile TypeScript into `dist`.
- `start`: run the compiled output from `dist`.
- `test`: run tests once.
- `test:watch`: run tests in watch mode.
- `lint`: run ESLint.
- `format`: format supported files with Prettier.
- `format:check`: check formatting without writing changes.

Recommended script shape:

```json
{
  "type": "module",
  "engines": {
    "node": ">=22"
  },
  "packageManager": "pnpm@<version-from-local-pnpm>",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint .",
    "format": "prettier . --write",
    "format:check": "prettier . --check"
  }
}
```

Prisma scripts are intentionally deferred. They become active when Prisma is installed and configured in a later phase.

## TypeScript Configuration

`tsconfig.json` should:

- Set `rootDir` to `src`.
- Set `outDir` to `dist`.
- Enable strict type checking.
- Include Node-compatible module resolution.
- Exclude generated output and dependencies.
- Include test files only if the selected structure requires type checking them during `npm run build`.

Recommended baseline:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["dist", "node_modules"]
}
```

## Environment Example

`.env.example` should document values expected by the HLD and architecture docs:

```dotenv
NODE_ENV=development
PORT=4000
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/next_blog
REDIS_URL=redis://localhost:6379
SESSION_SECRET=replace-with-a-long-random-secret
CORS_ORIGIN=http://localhost:3000
COOKIE_SECURE=false
```

Do not commit real secrets. `SESSION_SECRET` must remain a placeholder.

## Implementation Steps

1. Confirm `next-blog-server` does not already contain a Node package setup.
2. Initialize `package.json` with `pnpm init`.
3. Install the selected TypeScript and test dependencies.
4. Add the agreed scripts to `package.json`.
5. Add `.nvmrc` with Node `22`.
6. Create `tsconfig.json` with strict compiler settings and `dist` output.
7. Create ESLint and Prettier configuration files.
8. Create a minimal `src/index.ts` entry point.
9. Create one smoke test that verifies Vitest runs.
10. Add `.env.example` with the required environment variables.
11. Add or update `.gitignore` for `node_modules`, `dist`, `.env`, coverage, and logs.
12. Run the Phase 0 verification commands and fix any setup issues.

## Minimal Verification

Run:

```bash
pnpm build
pnpm test
pnpm lint
pnpm format:check
```

Optional sanity checks:

```bash
pnpm dev
pnpm start
```

`pnpm dev` and `pnpm start` only need to prove that the entry point can execute. They do not need to start a complete API server until Phase 1.

## Exit Criteria

Phase 0 is complete when:

- `package.json` exists with the required scripts.
- `package.json` uses ESM and documents Node 22.
- `.nvmrc` exists with Node 22.
- `tsconfig.json` exists and `pnpm build` succeeds.
- The test stack is installed and `pnpm test` succeeds.
- ESLint and Prettier are configured, and `pnpm lint` plus `pnpm format:check` succeed.
- `.env.example` documents `DATABASE_URL`, `REDIS_URL`, `SESSION_SECRET`, `PORT`, `NODE_ENV`, `CORS_ORIGIN`, and cookie/session flags.
- Generated/build artifacts and secrets are ignored by git.
- Prisma scripts are not present yet; they are deferred until Prisma setup.
- No API behavior is claimed as complete.

## Handoff to Phase 1

Phase 1 can start after this phase by adding:

- The full `src/` folder layout from `backend_architecture.md`.
- Zod environment validation in `src/config/env.ts`.
- Express app construction and middleware order.
- Redis and Prisma configuration modules.
- Centralized fallback error handling for unexpected `500` errors and unmatched `/api/*` routes.
