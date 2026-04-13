# CLAUDE.md

## Project Overview

R2-Explorer is a Google Drive-like interface for Cloudflare R2 storage buckets. It is a **pnpm monorepo** containing four packages, published as a single npm package (`r2-explorer`, v1.1.16) that users deploy via Cloudflare Workers.

**Stack:** TypeScript + Hono (backend) · Vue 3 + Quasar (frontend) · Cloudflare Workers (runtime) · Vitest + Playwright (tests) · Biome (linting)

---

## Repository Structure

```
R2-Explorer/
├── packages/
│   ├── worker/          # Backend API — published npm package (r2-explorer)
│   ├── dashboard/       # Frontend SPA — bundled into worker at build time
│   ├── docs/            # Documentation site (VitePress)
│   └── github-action/  # GitHub Actions deployment helper
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

### Worker Tests (`packages/worker/tests/`)

**Framework:** Vitest + `@cloudflare/vitest-pool-workers`  
**Config:** `tests/vitest.config.mts` — uses Miniflare with local R2 buckets: `MY_TEST_BUCKET_1`, `MY_TEST_BUCKET_2`, `teste`

Integration test files:
- `integration/server.test.ts` — `/api/server/config` endpoint
- `integration/buckets.test.ts` — bucket listing
- `integration/object.test.ts` — object CRUD operations
- `integration/copy.test.ts` — copy operation
- `integration/multipart.test.ts` — multipart upload flow
- `integration/shareLinks.test.ts` — share link creation, access, expiration
- `integration/email.test.ts` — email endpoints
- `integration/dashboard.test.ts` — dashboard asset serving
- `integration/setup.ts` — shared test utilities

### Dashboard Tests (`packages/dashboard/`)

**Unit tests** (`tests/unit/`): `appUtils.test.ts`, `auth-store.test.ts`, `main-store.test.ts`, `routes.test.ts`  
**Component tests** (`tests/components/`): `FilesFolderPage`, `FileContextMenu`, `LeftSidebar`, `MainLayout`, `Topbar`, `FilePreview`, `LoginPage`, `BucketPicker`, `CreateFolder`

### E2E Tests (`packages/dashboard/e2e/`)

**Framework:** Playwright (Chromium only, workers: 1)  
**Web server:** `wrangler dev` on port 8787 using `packages/worker/dev/wrangler-e2e.toml`

Test files: `app-loads.spec.ts`, `navigation.spec.ts`, `file-browsing.spec.ts`, `file-upload.spec.ts`, `file-operations.spec.ts`, `file-preview.spec.ts`, `context-menu.spec.ts`, `metadata.spec.ts`, `share-links.spec.ts`, `email.spec.ts`

---

## Package Details

### Worker (`packages/worker/`)

**Entry point:** `src/index.ts`  
**Framework:** Hono + Chanfana (OpenAPI/Zod)  
**Main export:** `R2Explorer(config?: R2ExplorerConfig)`

**API Routes:**
| Method | Route | Handler | Purpose |
|--------|-------|---------|---------|
| `GET` | `/api/server/config` | `GetInfo` | Server configuration & auth status |
| `GET` | `/api/buckets/:bucket` | `ListObjects` | List objects (cursor-paginated) |
| `GET` | `/api/buckets/:bucket/:key` | `GetObject` | Download file |
| `HEAD` | `/api/buckets/:bucket/:key` | `HeadObject` | Object metadata |
| `GET` | `/api/buckets/:bucket/:key/head` | `HeadObject` | Metadata alias (HEAD workaround) |
| `POST` | `/api/buckets/:bucket/upload` | `PutObject` | Upload single file |
| `POST` | `/api/buckets/:bucket/:key` | `PutMetadata` | Update HTTP/custom metadata |
| `POST` | `/api/buckets/:bucket/delete` | `DeleteObject` | Delete objects |
| `POST` | `/api/buckets/:bucket/move` | `MoveObject` | Move/rename object |
| `POST` | `/api/buckets/:bucket/copy` | `CopyObject` | Copy/duplicate object |
| `POST` | `/api/buckets/:bucket/folder` | `CreateFolder` | Create folder |
| `POST` | `/api/buckets/:bucket/:key/share` | `CreateShareLink` | Create share link |
| `GET` | `/api/buckets/:bucket/shares` | `ListShares` | List share links |
| `DELETE` | `/api/buckets/:bucket/share/:shareId` | `DeleteShareLink` | Delete share link |
| `GET` | `/share/:shareId` | `GetShareLink` | Public file access (no auth) |
| `POST` | `/api/buckets/:bucket/multipart/create` | `CreateUpload` | Initiate multipart upload |
| `POST` | `/api/buckets/:bucket/multipart/upload` | `PartUpload` | Upload a part |
| `POST` | `/api/buckets/:bucket/multipart/complete` | `CompleteUpload` | Finalize multipart upload |
| `POST` | `/api/emails/send` | `SendEmail` | Send email (MailChannels) |

**Middleware chain:** CORS → ReadOnly → Authentication → Routes

**Authentication:** Basic Auth (`basicAuth` middleware) or Cloudflare Access (`@hono/cloudflare-access`). Both configured via `R2ExplorerConfig`.

**Read-only mode:** Default `true`. Blocks write operations. Disable via `R2Explorer({ readonly: false })`.

**Module layout:**
```
src/
├── index.ts                          # Route registration, app factory
├── types.d.ts                        # All TypeScript types
├── foundation/
│   ├── settings.ts                   # Package version
│   ├── dates.ts                      # Date utilities
│   └── middlewares/readonly.ts       # Blocks mutations in read-only mode
└── modules/
    ├── dashboard.ts                  # ASSETS binding handlers
    ├── server/getInfo.ts             # Server config endpoint
    ├── buckets/                      # All bucket operations (one file per endpoint)
    │   ├── multipart/                # createUpload, partUpload, completeUpload
    │   └── ...
    └── emails/
        ├── sendEmail.ts              # Email send
        └── receiveEmail.ts           # Cloudflare Email Routing handler
```

**TypeScript types** (`src/types.d.ts`):

```typescript
type R2ExplorerConfig = {
  readonly?: boolean;              // Default: true (blocks mutations)
  cors?: boolean;                  // Enable CORS on /api/*
  cfAccessTeamName?: string;       // Cloudflare Access team name
  dashboardUrl?: string;
  emailRouting?: { targetBucket: string } | false;
  showHiddenFiles?: boolean;
  basicAuth?: BasicAuthType | BasicAuthType[];
  buckets?: Record<string, BucketConfig>;
};

type ShareMetadata = {
  bucket: string; key: string;
  expiresAt?: number; passwordHash?: string;
  maxDownloads?: number; currentDownloads: number;
  createdBy: string; createdAt: number;
};

type AppEnv = { ASSETS: Fetcher; [key: string]: R2Bucket };
type AppVariables = {
  config: R2ExplorerConfig;
  authentication_type?: string;
  authentication_username?: string;
} & CloudflareAccessVariables;
```

**Key dependencies:** `hono` (^4.10.7), `chanfana` (^2.8.3), `@hono/cloudflare-access` (^0.3.1), `postal-mime` (^2.6.1), `zod` (^3.25.76)

### Dashboard (`packages/dashboard/`)

Vue 3 SPA built with Quasar. Compiled to `dist/spa/`, then copied into `packages/worker/dashboard/` during `pnpm build-worker`. Served via Cloudflare Workers Assets binding.

**Key directories:**
```
src/
├── components/
│   ├── main/         # Topbar, LeftSidebar, RightSidebar, BucketPicker
│   ├── files/        # CreateFile, CreateFolder, FileOptions, ShareFile
│   ├── preview/      # FilePreview, PdfViewer, EmailViewer, logGz
│   └── utils/        # DragAndDrop, uploadingPopup
├── pages/
│   ├── files/        # FilesFolderPage (main browser), FileContextMenu
│   └── email/        # EmailFolderPage, EmailFilePage
├── layouts/          # MainLayout, AuthLayout
├── stores/           # main-store.js (config/buckets), auth-store.js (login/logout)
├── router/           # routes.js (6 route groups)
├── boot/             # axios.js, auth.js, bus.js
├── parsers/          # markdown.js
└── appUtils.js       # All API fetch calls (see below)
```

**Router routes** (`src/router/routes.js`):
```
/                                  → HomePage (redirect)
/:bucket/files                     → FilesFolderPage
/:bucket/files/:folder             → FilesFolderPage (nested)
/:bucket/files/:folder/:file       → FilesFolderPage (with file preview)
/:bucket/email                     → EmailFolderPage
/:bucket/email/:folder/:file       → EmailFilePage
/auth/login                        → LoginPage
```

**`appUtils.js` API functions** (centralized API client):
- `apiHandler.listObjects()`, `fetchFilePage()`, `fetchFile()` — listing
- `apiHandler.uploadObjects()`, `multipartCreate/Upload/Complete()` — uploads (with 5-attempt retry backoff)
- `apiHandler.downloadFile()`, `headFile()` — file access
- `apiHandler.renameObject()`, `copyObject()`, `deleteObject()` — mutations
- `apiHandler.updateMetadata()` — metadata editing
- `apiHandler.createShareLink()`, `listShares()`, `deleteShareLink()` — sharing
- Helpers: `encode()`, `decode()`, `sleep()`, `retryWithBackoff()`, `timeSince()`, `bytesToSize()`

**Key dependencies:** `vue` (^3.3.11), `quasar` (^2.17.5), `pinia` (^2.0.11), `axios` (^1.2.1), `vue-router` (^4.2.5), `pdfjs-dist` (2.5.207)

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

Notable Biome disabled rules: `noExplicitAny`, `noParameterAssign`, `noForEach`, `useLiteralKeys`, `noDelete`.

---

## Important Files

| File | Purpose |
|------|---------|
| `packages/worker/src/index.ts` | Worker entry — route registration, auth middleware |
| `packages/worker/src/types.d.ts` | TypeScript types (Config, Context, ShareMetadata) |
| `packages/worker/src/foundation/settings.ts` | Package version |
| `packages/worker/src/foundation/middlewares/readonly.ts` | Read-only mode enforcement |
| `packages/worker/src/modules/buckets/copyObject.ts` | Copy/duplicate objects |
| `packages/worker/src/modules/buckets/createShareLink.ts` | Share link creation logic |
| `packages/worker/src/modules/buckets/getShareLink.ts` | Public share access |
| `packages/worker/src/modules/emails/receiveEmail.ts` | Cloudflare Email Routing handler |
| `packages/dashboard/src/appUtils.js` | All API fetch calls from the dashboard |
| `packages/dashboard/src/pages/files/FilesFolderPage.vue` | Main file browser UI |
| `packages/dashboard/src/pages/files/FileContextMenu.vue` | Right-click context menu |
| `packages/dashboard/src/components/files/ShareFile.vue` | Share link UI |
| `packages/dashboard/src/components/utils/DragAndDrop.vue` | Upload drag-and-drop |
| `packages/dashboard/src/stores/main-store.js` | Global state (buckets, config) |
| `packages/dashboard/src/stores/auth-store.js` | Auth state (login/logout) |
| `template/src/index.ts` | End-user entry point example |
| `template/wrangler.toml` | End-user Wrangler config example |
| `biome.json` | Linting/formatting rules |
| `pnpm-workspace.yaml` | Workspace package definitions |

---

## Share Links

Share metadata is stored in R2 under the `.r2-explorer/sharable-links/` prefix. Passwords are hashed with SHA-256. The public `/share/:shareId` endpoint bypasses authentication and checks expiration + download limits on every request.

---

## Email Integration

- **Receiving:** `receiveEmail` handler (exported from `R2Explorer` return value as `email`) integrates with Cloudflare Email Routing. Emails are parsed with `postal-mime` and stored in R2.
- **Sending:** `POST /api/emails/send` via MailChannels (currently a stub).
- **Config:** `emailRouting: { targetBucket: "BUCKET_BINDING" }` or `false` to disable.
- **Wrangler:** Add `[[email_routing]]` binding to expose the `email` export.

---

## Build Pipeline

1. `pnpm build-dashboard` → Quasar compiles Vue SPA to `packages/dashboard/dist/spa/`
2. `pnpm build-worker` → tsup compiles TS, copies dashboard dist to `packages/worker/dashboard/`, copies README + LICENSE + docs
3. `pnpm package` → Creates npm tarball from `packages/worker/`
4. `pnpm release` (CI only) → Publishes to npm via Changesets

---

## Environment Bindings (`wrangler.toml`)

```toml
[[r2_buckets]]
binding = "MY_BUCKET"           # env.MY_BUCKET in Worker code
bucket_name = "actual-bucket"

[assets]
directory = "node_modules/r2-explorer/dashboard"
binding = "ASSETS"
html_handling = "auto-trailing-slash"
not_found_handling = "single-page-application"
```

---

## Debugging

**Worker:**
- Wrangler logs: `wrangler tail`
- Verify R2 bucket bindings in `wrangler.toml`
- Object keys are base64-encoded in routes; see `encode()`/`decode()` in `appUtils.js`

**Dashboard:**
- Browser console for API errors
- Quasar dev mode: `cd packages/dashboard && pnpm dev`
- Auth token stored in `sessionStorage` or `localStorage`

**Build:**
- Clear outputs: `rm -rf packages/*/dist`
- Reinstall: `rm -rf node_modules && pnpm install`

---

## Security Considerations

1. **Authentication:** Always enable Basic Auth or Cloudflare Access in production.
2. **Read-only mode:** Keep enabled unless write access is required.
3. **CORS:** Only enable if external API access is needed.
4. **Secrets:** Use Wrangler secrets (`wrangler secret put`) — never hardcode credentials.
5. **Share links:** Public `/share/:shareId` bypasses auth; enforce expiration and download limits for sensitive files.
6. **Content-Disposition:** `getObject.ts` sanitizes filenames to prevent header injection.

---

## Resources

- Docs: https://r2explorer.com
- Demo: https://demo.r2explorer.com
- Hono: https://hono.dev
- Chanfana (OpenAPI): https://chanfana.pages.dev
- Quasar: https://quasar.dev
- Cloudflare R2: https://developers.cloudflare.com/r2/
