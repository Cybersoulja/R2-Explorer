# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

R2-Explorer is a Google Drive-like interface for Cloudflare R2 storage buckets — a **pnpm monorepo** published as a single npm package (`r2-explorer`) that end users deploy via Cloudflare Workers.

**Stack:** TypeScript + Hono (backend) · Vue 3 + Quasar (frontend) · Cloudflare Workers (runtime) · Vitest + Playwright (tests) · Biome (linting)

---

## Commands

```bash
# Linting (run before every commit)
pnpm lint

# Build
pnpm build                             # Full: dashboard then worker
pnpm build-dashboard                   # Vue SPA → packages/dashboard/dist/spa/
pnpm build-worker                      # Compile TS + bundle dashboard into worker

# Testing
pnpm test                              # Worker + dashboard unit/component tests
pnpm test:e2e                          # Playwright E2E tests (builds dashboard first)
cd packages/worker && pnpm test        # Worker integration tests only
pnpm --filter r2-explorer-dashboard test  # Dashboard tests only

# Run a single worker test file
cd packages/worker && npx vitest run --config tests/vitest.config.mts tests/integration/object.test.ts

# Development
cd packages/dashboard && pnpm dev      # Dashboard dev server (proxies API to localhost:8787)
cd packages/worker && npx wrangler dev --config dev/wrangler.toml  # Worker dev server

# Release
pnpm changeset                         # Changeset for user-facing changes
pnpm changeset --empty                 # Changeset for internal-only PRs (required for every PR)
```

---

## Git Workflow

- **Never commit to `main` directly.** Branch naming: `feat/`, `fix/`, `chore/`.
- **Every PR needs a changeset** — user-facing: `pnpm changeset`; internal: `pnpm changeset --empty`.

---

## Architecture

### How the packages relate

The **worker** (`packages/worker/`) is the published npm package (`r2-explorer`). It is a Hono app that:
1. Serves the dashboard SPA via a Cloudflare Workers Assets binding (`ASSETS`)
2. Exposes a REST API at `/api/*` backed by Chanfana (OpenAPI + Zod)
3. Exports an `email` handler for Cloudflare Email Routing

The **dashboard** (`packages/dashboard/`) is a Vue 3 + Quasar SPA. During `pnpm build-worker`, its compiled output (`dist/spa/`) is copied into `packages/worker/dashboard/` and bundled into the published package. In production there is no separate frontend server — the worker serves all assets.

In **development**, the dashboard dev server (`pnpm dev`) proxies `/api/*` to `localhost:8787`, where `wrangler dev` runs the worker locally. The `dev/wrangler.toml` points directly at the dashboard `dist/spa/` directory.

### Worker request lifecycle

```
Request → CORS (optional) → ReadOnly middleware → Auth middleware → Routes
```

- **ReadOnly middleware** (`foundation/middlewares/readonly.ts`): blocks all non-GET/HEAD methods when `readonly: true` (the default).
- **Auth**: either `basicAuth` (Hono built-in) or Cloudflare Access (`@hono/cloudflare-access`). Both are optional and configured via `R2ExplorerConfig`. Auth username is stored in Hono context variables for use by handlers.
- **Routes**: each endpoint is a class extending `OpenAPIRoute` from Chanfana, one file per endpoint under `src/modules/`. Route registration order matters — the catch-all object routes (`GET /api/buckets/:bucket/:key`, `POST /api/buckets/:bucket/:key`) must be registered last.

### Object key encoding

All object keys in API routes are **base64-encoded**. The dashboard's `encode()`/`decode()` helpers in `appUtils.js` handle this. The root folder is represented as `"IA=="` (base64 of a space character) — stored as `ROOT_FOLDER`. Handlers decode keys with `decodeURIComponent(escape(atob(key)))`.

### Share links

Metadata is stored directly in R2 under `.r2-explorer/sharable-links/<shareId>.json` in the same bucket as the shared file. The public `/share/:shareId` endpoint bypasses auth middleware entirely (it is registered outside the `/api/*` prefix). Passwords are hashed with SHA-256 via the Web Crypto API.

### Dashboard data flow

- **`appUtils.js`** is the sole API client layer — all axios calls go through it.
- **`main-store.js`** (Pinia) holds server config, bucket list, readonly flag, and auth info. It is populated on boot by calling `/api/server/config`.
- **`auth-store.js`** manages Basic Auth credentials persisted in `sessionStorage`/`localStorage`.
- **Boot sequence** (`boot/auth.js`): check stored credentials → if none, hit `/api/server/config` unauthenticated → if 401, redirect to `/auth/login`.
- The axios `baseURL` is `{origin}/api` in production and `http://localhost:8787/api` in development (override with `VUE_APP_SERVER_URL`).

### Adding a new API endpoint

1. Create `packages/worker/src/modules/<feature>/myEndpoint.ts` extending `OpenAPIRoute`.
2. Define Zod schema for `request.params`, `request.query`, or `request.body`.
3. In `handle(c)`, call `this.getValidatedData<typeof this.schema>()` to get typed params.
4. Access the R2 bucket via `c.env[bucketName]` — throw `HTTPException(500)` if missing.
5. Register in `packages/worker/src/index.ts` (before the catch-all object routes).
6. Add integration tests in `packages/worker/tests/integration/`.
7. If there's a UI, add the API call to `packages/dashboard/src/appUtils.js`.

---

## Testing

### Worker tests (`packages/worker/tests/`)

Uses `@cloudflare/vitest-pool-workers` with Miniflare. The config at `tests/vitest.config.mts` spins up three local R2 buckets: `MY_TEST_BUCKET_1`, `MY_TEST_BUCKET_2`, `teste`. Use `createTestApp()` and `createTestRequest()` from `integration/setup.ts` to build test requests.

### Dashboard tests (`packages/dashboard/`)

Unit tests in `tests/unit/`, component tests in `tests/components/` using Vitest + `@vue/test-utils` + `happy-dom`.

### E2E tests (`packages/dashboard/e2e/`)

Playwright (Chromium only, single worker). Spins up `wrangler dev` on port 8787 via `packages/worker/dev/wrangler-e2e.toml`.

---

## Code Style

- **Linter/Formatter:** Biome. Run `pnpm lint` to check and auto-fix.
- **TypeScript:** Strict mode. All shared types in `packages/worker/src/types.d.ts`.
- **Vue:** Composition API with Quasar components.
- **Indentation:** Tabs in JS/TS, 2 spaces in YAML.
- **Quotes:** Double quotes.

Notable disabled Biome rules: `noExplicitAny`, `noParameterAssign`, `noForEach`, `useLiteralKeys`, `noDelete`.
