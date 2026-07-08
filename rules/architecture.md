---
paths:
  - "packages/**/*"
---

# Architecture, Data Model & Dependencies

Reference for the request pipeline, IPC / store / serialization internals, the core data-model
types, and dependency versions.

## Monorepo Structure (npm workspaces)

```
packages/
  bruno-app/          React renderer (rsbuild, Redux, styled-components)
  bruno-electron/     Electron main process (electron-builder)
  bruno-js/           JS sandbox for user scripts
  bruno-common/       Shared utilities (rollup)
  bruno-converters/   Format conversion (Postman, Insomnia, etc.)
  bruno-requests/     HTTP request execution engine (rollup)
  bruno-filestore/    .bru/.yml file serialization (rollup)
  bruno-query/        JSONPath query engine (rollup)
  bruno-lang/         Bruno DSL parser
  bruno-schema/       Collection/request schema validation
  bruno-schema-types/ TypeScript type definitions
  bruno-graphql-docs/ GraphQL documentation generator
  bruno-toml/         TOML parser
  bruno-cli/          Command-line runner
  bruno-tests/        Test servers (express: HTTP/HTTPS/proxy/GraphQL)
tests/                Playwright e2e tests
playwright/           Test fixtures and helpers (index.ts)
```

## Electron IPC Pattern

- **Main process entry**: `packages/bruno-electron/src/index.js`
- **IPC handlers**: `packages/bruno-electron/src/ipc/` (collection, workspace, network, git, preferences, etc.)
- **Persistent stores**: `packages/bruno-electron/src/store/` (electron-store backed: preferences, cookies, oauth2, etc.)
- **File watchers**: `packages/bruno-electron/src/app/` (collection-watcher, workspace-watcher, gitWatcher, dotenv-watcher)
- **Preload** (`preload.js`): Bridges main/renderer

See `.claude/rules/electron-ipc.md` for the startup sequence.

## Request Execution Pipeline

1. User triggers request in UI
2. `ipc/network/index.js` handles the IPC call
3. Variables interpolated (`@usebruno/common`)
4. Pre-request scripts run via `ScriptRuntime` (QuickJS or NodeVM)
5. HTTP request sent via axios (or gRPC/WebSocket client)
6. Post-response scripts and tests run
7. Assertions evaluated via `AssertRuntime`
8. Results sent back to renderer

## Redux Store (app side)

Slices and middleware live in `packages/bruno-app/src/providers/ReduxStore/` — read `slices/`
for the current set (e.g. `collections/`, `app`, `tabs`, `workspaces`, plus paid-only `git/`,
`license`, `trial`). Conventions in `.claude/rules/redux-store.md`.

## File Format System

- `bruno-filestore`: Central parser/serializer
- Two formats: `.bru` (custom Bru language) and `.yml` (OpenCollection YAML)
- `bruno-lang`: Bru parser using `arcsecond` and `ohm-js`
- Collections stored on filesystem, watched by chokidar

On-disk format changes → `.claude/rules/dsl-changes.md`.

## Script Sandboxing (`bruno-js`)

- **QuickJS** (default, safe mode): WebAssembly sandbox
- **Node VM** (developer mode): Node.js VM with more capabilities
- Scripts access `bru`, `req`, `res`, `test`, `expect`, `assert`, `console` objects

## Providers (`bruno-app/src/providers/`)

Read the directory for the current set — e.g. App, Theme, Hotkeys, ReduxStore, Toaster, plus
paid-only `LicenseGuard/` (premium gating) and `Dictionary/` (i18n).

## Key Types (`bruno-schema-types`)

Core data-model types live in `packages/bruno-schema-types/src/` — read them for the current
shapes and fields. The central ones: `Collection` (top-level
container of items / environments / config), `Item` (a request or folder, discriminated by
`type`), `HttpRequest`, `Auth` (a `mode` union), `KeyValue` (headers / params / assertions),
`Environment`, `Script`.

## Key Dependencies

Versions live in the root and per-package `package.json`. Broadly: React + Redux Toolkit + Styled Components + Tailwind + Rsbuild + Codemirror
(frontend); Electron + electron-builder + chokidar (desktop); axios + @grpc/grpc-js + ws (HTTP);
arcsecond + ohm-js + js-yaml (parsing); Rollup + TypeScript (build); Jest + Playwright + Chai (test).
