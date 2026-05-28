# Readest — Project Context for LLM/Agent Sessions

> **Last updated:** 2026-05-28  
> **Repository:** https://github.com/readest/readest  
> **License:** AGPL-3.0  
> **Monorepo root:** `apps/readest-app/` contains the single application codebase.

---

## 1. What This Project Is

**Readest** is an open-source, cross-platform ebook reader designed for immersive and deep reading experiences. It is a modern rewrite of [Foliate](https://github.com/johnfactotum/foliate), built as a hybrid **Next.js 16 + Tauri v2** application that compiles from a single TypeScript/React codebase into multiple targets:

- **Desktop app** — Windows, macOS, Linux (Tauri v2)
- **Mobile app** — iOS, Android (Tauri v2 mobile)
- **Web app** — deployed to Cloudflare Workers at [web.readest.com](https://web.readest.com) via OpenNext
- **Browser extension** — "Send to Readest"
- **Windows shell extension** — thumbnail provider

Supported book formats: **EPUB, MOBI, KF8 (AZW3), FB2, CBZ, TXT, PDF**.

---

## 2. Technology Stack

| Layer | Technologies |
|-------|-------------|
| **Frontend** | Next.js 16, React 19, TypeScript (strict, ES2022), TailwindCSS 3, DaisyUI 4 |
| **State** | Zustand 5 (single-purpose stores per domain) |
| **Native shell** | Tauri v2 (Rust 1.77.2+) |
| **Book engine** | `foliate-js` (forked, in `packages/foliate-js`), `pdfjs-dist` |
| **Database** | Turso/libSQL — WASM replica in browser, native plugin in Tauri |
| **Auth** | Supabase Auth |
| **Object storage** | S3-compatible (R2/S3) for cloud library |
| **Payments** | Stripe (web), Apple IAP, Google Play IAP |
| **AI / RAG** | Vercel AI SDK, OpenAI-compatible providers, Ollama |
| **Translation** | DeepL, Google, Azure, Yandex |
| **TTS** | Web Speech API, Native TTS (Tauri plugin), Edge TTS |
| **i18n** | i18next, react-i18next |
| **Testing** | Vitest (jsdom + browser + Tauri), Playwright, WebDriverIO |
| **Lint/Format** | Biome, `tsgo` (type check), Cargo fmt/clippy |
| **Package manager** | pnpm 11.1.1 |

---

## 3. Repository Structure

This is a **pnpm monorepo**. The vast majority of code lives under `apps/readest-app/`.

```
readest/
├── apps/readest-app/           # Main application (Next.js + Tauri)
│   ├── src/
│   │   ├── app/                # Next.js App Router (pages + API routes)
│   │   │   ├── api/            # App Router API endpoints (ai, stripe, tts, share, etc.)
│   │   │   ├── auth/           # Supabase auth UI
│   │   │   ├── library/        # Library grid, import, OPDS shelf
│   │   │   ├── reader/         # Reader UI (~80 components, ~30 hooks)
│   │   │   ├── opds/           # OPDS catalog browser
│   │   │   ├── send/           # Send-to-Readest inbox
│   │   │   ├── user/           # Account, subscription, settings
│   │   │   └── runtime-config.js/  # Server-injected runtime config endpoint
│   │   ├── pages/              # Next.js Pages Router (legacy)
│   │   │   ├── api/            # Pages Router API endpoints (sync, storage, send, deepl)
│   │   │   ├── reader/[ids].tsx# Historical reader entrypoint
│   │   │   └── _document.tsx   # COOP/COEP document shell
│   │   ├── components/         # Shared React UI primitives
│   │   ├── services/           # Business logic & platform adapters (see §4)
│   │   ├── store/              # Zustand stores
│   │   ├── hooks/              # Custom React hooks
│   │   ├── utils/              # Pure utility functions
│   │   ├── libs/               # Shared libraries (payment, storage, sync)
│   │   ├── types/              # Shared TypeScript types
│   │   ├── context/            # React Context providers
│   │   ├── workers/            # Browser Web Workers
│   │   ├── sw.ts               # Serwist service worker (PWA/offline)
│   │   └── middleware.ts       # CORS + COOP/COEP headers
│   ├── src-tauri/              # Rust Tauri backend
│   │   ├── src/
│   │   │   ├── lib.rs          # Command registration, deep links, scope grants
│   │   │   ├── dir_scanner.rs  # Recursive directory scanning
│   │   │   ├── transfer_file.rs# Chunked file upload/download
│   │   │   ├── discord_rpc.rs  # Discord Rich Presence (desktop)
│   │   │   └── {android,ios,macos,windows}/  # Platform glue
│   │   ├── plugins/            # Custom Tauri plugins
│   │   └── tauri.conf.json     # Tauri configuration
│   ├── extension/              # Browser extension code
│   ├── extensions/             # Platform integrations (Windows thumbnail)
│   ├── workers/                # Cloudflare Workers (e.g., send-email)
│   ├── public/                 # Static assets
│   └── docs/                   # App-specific docs (architecture.md, code-layout.md, testing.md)
├── packages/
│   └── foliate-js/             # Forked book rendering engine (git submodule)
├── Cargo.toml                  # Rust workspace definition
├── package.json                # Root monorepo scripts
└── .github/workflows/          # CI/CD (release, docker, PR checks)
```

### Path Aliases

- `@/*` → `./src/*`
- `@/components/ui/*` → `./src/components/primitives/*`

---

## 4. Core Architecture

### 4.1 Three Runtimes

The system has three process boundaries:

1. **Browser / WebView** — React UI, shared domain layer (`src/services`), `foliate-js`, Turso WASM, service worker.
2. **Tauri Native Host (Rust)** — Command handlers in `src-tauri/src/lib.rs`, Tauri plugins (fs, dialog, http, native-tts, turso, etc.).
3. **Next.js Server** — API routes (App + Pages Router), middleware, runtime-config route. On web this runs as a Cloudflare Worker; on Tauri targets it runs locally but mostly hits the production deployment.

### 4.2 Platform Abstraction (`AppService`)

The most important abstraction is `src/services/appService.ts`. Every piece of code that touches "the platform" (filesystem, native dialogs, shell open, native TTS, IAP, deep links, etc.) goes through an `AppService` interface. Three implementations exist:

- **`nativeAppService.ts`** — Tauri desktop/mobile (`@tauri-apps/api invoke()`)
- **`webAppService.ts`** — Browser (File API, IndexedDB, `fetch()`)
- **`nodeAppService.ts`** — Node tooling, tests, CLI

`src/services/environment.ts` decides at runtime which implementation to mount based on `NEXT_PUBLIC_APP_PLATFORM` and runtime detection (`window`, Tauri injection). Callers do `const appService = useEnv().appService` and remain platform-agnostic.

The same pattern repeats for the database layer (`src/services/database`):
- `webDatabaseService` (browser via Turso WASM)
- `nativeDatabaseService` (Tauri via `tauri-plugin-turso`)
- `nodeDatabaseService` (Node, used by tests)

### 4.3 Frontend State (Zustand)

State is split across single-purpose Zustand stores in `src/store`:

- `libraryStore` — books, folders, selection, sort
- `bookDataStore` — per-book data (TOC, annotations, locations)
- `readerStore` — active views, layout, ribbon state
- `settingsStore` / `themeStore` — user settings and theming
- `aiChatStore` — AI chat sessions
- `transferStore` — in-flight uploads/downloads
- And 15+ others

### 4.4 Backend Routes

**App Router (`src/app/api/*`)** — newer endpoints:
- `ai/chat`, `ai/embed` — AI chat & embeddings
- `stripe/*` — billing
- `tts/edge` — Edge TTS streaming
- `metadata/search` — book metadata
- `opds/proxy` — CORS-friendly OPDS proxy
- `share/*` — share-link landing
- `apple/iap-verify`, `google/iap-verify` — IAP verification

**Pages Router (`src/pages/api/*`)** — legacy/sync/storage:
- `sync.ts`, `kosync.ts` — KOReader-compatible sync
- `sync/replicas.ts`, `sync/replica-keys.ts` — modern replica sync
- `storage/*` — presigned S3 upload/download/list/delete
- `send/*` — "Send to Readest" inbox
- `deepl/translate.ts` — DeepL proxy (hides API key)

### 4.5 Runtime Config

`src/app/runtime-config.js/route.ts` emits `window.__READEST_RUNTIME_CONFIG = {...}` as a JS file loaded from `layout.tsx` and `_document.tsx`. This lets a single Docker image be rebranded with different Supabase projects, S3 endpoints, or quotas at deploy time without rebuilding.

### 4.6 Cross-Origin Isolation (Critical)

`middleware.ts` sets `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` on every document. This is **required** so the browser exposes `SharedArrayBuffer`, which the Turso WASM thread pool needs. Without these headers, `initThreadPool` hangs.

### 4.7 Sync Architecture

Two sync paths coexist:

1. **Legacy KOReader-compatible sync** — `KOSyncClient.ts` against `pages/api/sync.ts` / `kosync.ts`.
2. **Modern replica sync** — Encrypts replicas locally with a passphrase-derived key, publishes deltas to `pages/api/sync/replicas.ts`, pulls peer updates, and applies them through category adapters in `src/services/sync/adapters/*`.

Cloud library (book bytes) is distinct from replica sync and uses the storage layer (`pages/api/storage/*`).

---

## 5. How to Run It

### Prerequisites

- Node.js v22+ and pnpm 11.1.1+
- Rust 1.77.2+ and Cargo
- For Windows: Build Tools for Visual Studio 2022 with "Desktop development with C++"

```bash
# 1. Clone and init submodules
git clone https://github.com/readest/readest.git
cd readest
git submodule update --init --recursive

# 2. Install dependencies
pnpm install
pnpm --filter @readest/readest-app setup-vendors

# 3. Verify Tauri setup
pnpm tauri info
```

### Development Commands

```bash
# Web-only dev (no Rust compilation)
pnpm dev-web

# Desktop dev with Tauri
pnpm tauri dev

# Build Next.js for Tauri
pnpm build

# Build for web deployment
pnpm build-web

# Mobile dev
pnpm tauri android dev
pnpm tauri ios dev
```

### Testing Commands

```bash
pnpm test                  # Unit tests (vitest + jsdom)
pnpm test:browser          # Browser tests (Chromium via Playwright)
pnpm test:tauri            # Tauri integration tests (requires app running with webdriver)
pnpm test:e2e              # E2E tests (WebDriverIO)
pnpm test:e2e:web          # Playwright E2E against web build
```

### Linting & Formatting

```bash
pnpm lint                  # Biome lint + tsgo type check + Lua lint
pnpm format                # Biome format (run from monorepo root)
pnpm fmt:check             # Rust format check
pnpm clippy:check          # Rust lint
```

### Git Worktrees

Always use the project's worktree script (handles submodules, `.env`, vendors, Tauri gen symlinks):

```bash
pnpm worktree:new feat/my-feature
pnpm worktree:new 3837      # Checkout PR #3837
```

---

## 6. Build & Deploy Targets

```
Source (apps/readest-app)
    ├── next build + @opennextjs/cloudflare → Cloudflare Workers (web.readest.com)
    ├── next build + Docker                 → Self-hostable Docker image (GHCR)
    ├── next build → tauri build            → Desktop (dmg / nsis / appimage)
    ├── next build → tauri android/ios build → Mobile (aab / ipa)
    └── Extension build                     → Browser extension package
```

Tauri builds use `dotenv` to switch env files (`.env.tauri`, `.env.web`, `.env.apple-*.local`, etc.) for platform-specific configuration.

---

## 7. Critical Implementation Details

### Security: `allow_paths_in_scopes` (Tauri)
In `src-tauri/src/lib.rs`, `allow_paths_in_scopes` extends `fs_scope` and `asset_protocol_scope` **only for paths the Tauri dialog plugin (or persisted-scope on restart) already granted**. Without this gate, frontend code (including XSS through book content or OPDS HTML) could read the user's home directory via the asset protocol.

### E-ink Mode
Every new UI widget must look correct under `[data-eink='true']`. E-ink screens have no shadows, no gradients, slow refresh, and need crisp 1px borders. Use `eink-bordered` for surfaces/inputs and `btn-primary` for primary actions. Do not rely on color/shadow alone for hierarchy.

### i18n Approach
Readest uses a **key-as-content** translation approach. See `docs/i18n.md` for `stubTranslation` usage in non-React modules and the extraction workflow (`pnpm i18n:extract`).

### Chinese Text Processing
- Conversion: `simplecc-wasm` (copied to `public/vendor/simplecc`)
- Segmentation: `jieba-wasm` (copied to `public/vendor/jieba`)

### PDF Rendering
PDF.js is copied into `public/vendor/pdfjs` at build time (`pnpm setup-pdfjs`). Includes worker, WASM, fonts, CMaps, and flattened CSS.

---

## 8. Key External Services

| Service | Purpose |
|---------|---------|
| Supabase | Auth + Postgres |
| S3 / R2 | Object storage for book bytes |
| Stripe | Web billing |
| Apple/Google | Mobile IAP |
| DeepL / Google / Azure / Yandex | Translation |
| OpenAI / Ollama / etc. | AI chat & embeddings |
| Edge TTS | Streaming Microsoft voices |
| Google Books / Open Library | Book metadata |
| Hardcover | Reading progress sync |
| Readwise | Highlights export |
| Wikipedia / Wiktionary | Online dictionary lookup |
| OPDS catalogs / Calibre | Online library access |

---

## 9. References

- **Detailed architecture:** `apps/readest-app/docs/architecture.md`
- **Code layout & runtime boundaries:** `apps/readest-app/docs/code-layout.md`
- **Testing guide:** `apps/readest-app/docs/testing.md`
- **Design system:** `apps/readest-app/DESIGN.md`
- **Agent workspace:** `apps/readest-app/AGENTS.md`
- **Contributing:** `CONTRIBUTING.md`
- **i18n:** `apps/readest-app/docs/i18n.md`
- **Safe area insets:** `apps/readest-app/docs/safe-area-insets.md`
