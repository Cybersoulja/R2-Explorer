# CLAUDE.md

## Project Overview

R2-Explorer is a Google Drive-like interface for Cloudflare R2 storage buckets. It is a **pnpm monorepo** containing four packages, published as a single npm package (`r2-explorer`) that users deploy via Cloudflare Workers.

**Stack:** TypeScript + Hono (backend) · Vue 3 + Quasar (frontend) · Cloudflare Workers (runtime) · Vitest + Playwright (tests) · Biome (linting)

---

## Repository Structure

```
R2-Explorer/
├── packages/
│   ├── worker/          # Backend API — published npm package (r2-explorer)
│   ├── dashboard/       # Frontend SPA — bundled into worker at build time
│   ├── docs/            # Documentation site (VitePress)
│   └── github-action/   # GitHub Actions deployment helper
├── template/            # Starter template for end users
├── biome.json           # Linting/formatting config (replaces ESLint + Prettier)
├── pnpm-workspace.yaml  # Monorepo workspace definition
└── package.json         # Root scripts
```

---

## Git Workflow

- **Never commit directly to `main`.** Always create a feature branch.
- **Run `pnpm lint` before every commit.** Fix any lint errors before committing.
- Branch naming: `feat/description`, `fix/description`, `chore/description`

---

## Commands Reference

```bash
# Linting
pnpm lint                              # Biome check + auto-fix

# Building
pnpm build                             # Build dashboard then worker (full)
pnpm build-dashboard                   # Build Vue SPA to packages/dashboard/dist/spa/
pnpm build-worker                      # Compile TS + bundle dashboard into worker

# Testing
pnpm test                              # Run worker + dashboard tests
pnpm test:e2e                          # Playwright E2E tests
pnpm --filter r2-explorer-dashboard test  # Dashboard component tests only

# Development
cd packages/dashboard && pnpm dev      # Dashboard dev server
cd packages/worker && pnpm test        # Worker integration tests

# Release
pnpm changeset                         # Create changeset for PR
pnpm changeset --empty                 # Empty changeset for internal-only PRs
pnpm release                           # Build + publish to npm (CI only)
```

---

## Changesets

Every PR must include a changeset:

- **User-facing changes** (new features, bug fixes, API changes): `pnpm changeset` — write a meaningful changelog entry with examples.
- **Internal-only changes** (refactoring, CI, docs, tooling): `pnpm changeset --empty`.

---

## Testing

- **UI changes must be covered by E2E tests.** Playwright tests live in `packages/dashboard/e2e/`. Run with `pnpm test:e2e`.
- Worker integration tests live in `packages/worker/tests/`. Run with `cd packages/worker && pnpm test`.
- Dashboard component tests: `pnpm --filter r2-explorer-dashboard test`.
- All tests: `pnpm test`.

---

## Package Details

### Worker (`packages/worker/`)

**Entry point:** `src/index.ts`  
**Framework:** Hono + Chanfana (OpenAPI/Zod)  
**Main export:** `R2Explorer(config?: R2ExplorerConfig)`

**API Routes:**
| Route | Purpose |
|-------|---------|
| `GET /api/server/config` | Server configuration |
| `GET /api/buckets/:bucket` | List objects |
| `GET /api/buckets/:bucket/:key` | Get object |
| `HEAD /api/buckets/:bucket/:key` | Head object |
| `POST /api/buckets/:bucket/upload` | Upload file |
| `POST /api/buckets/:bucket/:key` | Update metadata |
| `POST /api/buckets/:bucket/delete` | Delete objects |
| `POST /api/buckets/:bucket/move` | Move/rename object |
| `POST /api/buckets/:bucket/copy` | Copy object |
| `POST /api/buckets/:bucket/folder` | Create folder |
| `POST /api/buckets/:bucket/:key/share` | Create share link |
| `GET /api/buckets/:bucket/shares` | List share links |
| `DELETE /api/buckets/:bucket/share/:shareId` | Delete share link |
| `GET /share/:shareId` | Public file access (no auth) |
| `POST /api/buckets/:bucket/multipart/*` | Multipart upload |
| `POST /api/emails/send` | Send email |

**Middleware chain:** CORS → ReadOnly → Authentication → Routes

**Authentication:** Basic Auth (`basicAuth` middleware) or Cloudflare Access (`@hono/cloudflare-access`). Both configured via `R2ExplorerConfig`.

**Read-only mode:** Default `true`. Blocks write operations. Disable via `R2Explorer({ readonly: false })`.

### Dashboard (`packages/dashboard/`)

Vue 3 SPA built with Quasar. Compiled to `dist/spa/`, then copied into `packages/worker/dashboard/` during `pnpm build-worker`. Served via Cloudflare Workers Assets binding.

**Key directories:**
```
src/
├── components/   # Reusable Vue components
├── pages/        # Route-level pages
├── layouts/      # Page layouts (MainLayout, AuthLayout)
├── stores/       # Pinia state (main-store.js, auth-store.js)
├── router/       # Vue Router config
├── boot/         # Quasar boot files (axios, auth, bus)
└── appUtils.js   # API client (all fetch calls go here)
```

---

## Adding a New API Endpoint

1. Create a file in `packages/worker/src/modules/<feature>/myEndpoint.ts`.
2. Extend `OpenAPIRoute` from Chanfana and define a Zod schema.
3. Implement the `handle(c)` method.
4. Register the route in `packages/worker/src/index.ts`.
5. Add integration tests in `packages/worker/tests/integration/`.
6. If the endpoint has a UI, update `packages/dashboard/src/appUtils.js`.

```typescript
import { OpenAPIRoute } from "chanfana";
import { z } from "zod";

export class MyEndpoint extends OpenAPIRoute {
  schema = {
    request: {
      params: z.object({ bucket: z.string() }),
    },
    responses: { "200": { description: "Success" } },
  };

  async handle(c) {
    const { params } = await this.getValidatedData<typeof this.schema>();
    return c.json({ success: true });
  }
}
```

---

## Code Style

- **Linter/Formatter:** Biome (`biome.json`). Run `pnpm lint` to check and auto-fix.
- **TypeScript:** Strict mode. Types in `packages/worker/src/types.d.ts`.
- **Vue:** Composition API. Quasar components for UI consistency.
- **Imports:** Managed by Biome (auto-sorted on lint).
- **Quotes:** Double quotes in JS/TS.
- **Indentation:** Tabs (code), 2 spaces (YAML).

---

## Important Files

| File | Purpose |
|------|---------|
| `packages/worker/src/index.ts` | Worker entry — route registration, auth middleware |
| `packages/worker/src/types.d.ts` | TypeScript types (Config, Context, ShareMetadata) |
| `packages/worker/src/foundation/settings.ts` | Package version |
| `packages/dashboard/src/appUtils.js` | All API fetch calls from the dashboard |
| `packages/dashboard/src/pages/files/FilesFolderPage.vue` | Main file browser UI |
| `packages/dashboard/src/components/files/ShareFile.vue` | Share link UI |
| `packages/dashboard/src/components/utils/DragAndDrop.vue` | Upload drag-and-drop |
| `template/src/index.ts` | End-user entry point example |
| `template/wrangler.toml` | End-user Wrangler config example |
| `biome.json` | Linting/formatting rules |
| `pnpm-workspace.yaml` | Workspace package definitions |

---

## Share Links

Share metadata is stored in R2 under the `.r2-explorer/sharable-links/` prefix. Passwords are hashed with SHA-256. The public `/share/:shareId` endpoint bypasses authentication and checks expiration on every request.

---

## Build Pipeline

1. `pnpm build-dashboard` → Quasar compiles Vue SPA to `packages/dashboard/dist/spa/`
2. `pnpm build-worker` → tsup compiles TS, copies dashboard dist to `packages/worker/dashboard/`, copies README + LICENSE
3. `pnpm package` → Creates npm tarball from `packages/worker/`
4. `pnpm release` (CI only) → Publishes to npm via Changesets

---

## Resources

- Docs: https://r2explorer.com
- Demo: https://demo.r2explorer.com
- Hono: https://hono.dev
- Chanfana (OpenAPI): https://chanfana.pages.dev
- Quasar: https://quasar.dev
- Cloudflare R2: https://developers.cloudflare.com/r2/
