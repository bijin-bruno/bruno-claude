---
paths:
  - "packages/**/*"
---

# Architecture, Data Model & Dependencies

Reference for the monorepo layout, dependency directions, the request pipeline, the sandbox, the
core data-model types, and dependency versions. The area-specific rules
(`electron-ipc.md`, `redux-store.md`, `dsl-changes.md`) go deeper; this file is the map.

## Monorepo structure (npm workspaces)

```
packages/
  bruno-app/          React renderer (rsbuild, Redux Toolkit, styled-components)   [name @usebruno/app]
  bruno-electron/     Electron main process (electron-builder)                     [name "bruno" — NOT scoped]
  bruno-js/           JS sandbox for user scripts (QuickJS + node:vm)
  bruno-common/       Shared utilities — the base leaf (rollup → dist/cjs+esm)
  bruno-converters/   Import/export (Postman, Insomnia, OpenAPI, …) (rollup)
  bruno-requests/     Shared HTTP/gRPC/WS request building blocks (rollup)
  bruno-filestore/    .bru/.yml serialization (rollup + tsc --emitDeclarationOnly)
  bruno-query/        JSONPath-style query engine (rollup)
  bruno-lang/         Bru DSL grammars — v1 (arcsecond, legacy) + v2 (ohm-js, current)
  bruno-schema/       Yup runtime validation of collections/requests
  bruno-schema-types/ TypeScript type definitions only (tsc, types-only)
  bruno-graphql-docs/ GraphQL documentation generator (rollup)
  bruno-toml/         Thin wrapper over @iarna/toml (orphaned — on no live data path)
  bruno-cli/          Command-line runner (run straight from src/)
  bruno-tests/        Test servers (express: HTTP/HTTPS/proxy/GraphQL)
  bruno-docs/         @usebruno/docs — placeholder stub (only package.json + readme, no src/)
tests/                Playwright e2e tests
playwright/           Test fixtures and helpers (index.ts)
```

Build tool per package (matters for the "rebuild shared packages" step in `.claude/CLAUDE.md`):
- **rollup** (emits `dist/cjs` + `dist/esm`): bruno-common, bruno-converters, bruno-requests,
  bruno-query, bruno-graphql-docs; **bruno-filestore** additionally runs `tsc --emitDeclarationOnly`.
- **tsc only**: bruno-schema-types.
- **rsbuild**: bruno-app.
- **no build step (consumed straight from `src/`)**: bruno-js, bruno-lang, bruno-schema,
  bruno-toml, bruno-cli, bruno-electron, bruno-tests, bruno-docs.

The 7 packages that must be rebuilt after editing (they emit `dist/`): bruno-common,
bruno-requests, bruno-filestore, bruno-converters, bruno-query, bruno-graphql-docs,
bruno-schema-types. Editing bruno-js / bruno-lang / bruno-schema / bruno-toml needs no rebuild.

## Dependency direction & ownership boundaries

Internal `@usebruno/*` dependencies (from each `package.json`) form a strict DAG. **New code must
respect it** — a cyclic or upward dependency is an architectural bug, not a convenience.

- **Leaf libs — zero internal `@usebruno/*` deps:** bruno-common, bruno-lang, bruno-query,
  bruno-requests, bruno-graphql-docs, bruno-schema, bruno-schema-types, bruno-toml.
- **Mid consumers:** bruno-js → (common, query); bruno-converters → (common, schema; schema-types
  as devDep); bruno-filestore → (common, lang; schema-types as devDep).
- **Top consumers (things flow *into* them, never out):** bruno-cli → (common, converters,
  filestore, js, lang, requests); bruno-electron → (common, converters, filestore, js, lang,
  requests, schema); bruno-app → (common, converters, graphql-docs, schema).

Guardrails this enforces:

1. **bruno-common is the browser-safe base leaf.** It runs in the web renderer (`bruno-app`), not
   just Node, so it must stay platform-neutral: no Node built-ins (`fs`, `path`, `os`, `crypto`,
   `child_process`, `node:*`) and no dependency that itself pulls in Node. It currently ships with
   zero runtime dependencies — keep it that way. It also depends on no other `@usebruno/*` package;
   a util that needs Node or another bruno package belongs in a different package.
2. **No shared/library package may import bruno-app or bruno-electron.** Renderer- or
   Electron-specific code must not be pushed down into a lib to make it importable upward.
3. **bruno-js must stay Electron-free.** It runs in both the Electron main process *and* the CLI
   (bruno-cli → js). Adding an `electron`/IPC import to bruno-js breaks the CLI. Sandbox logic
   belongs in bruno-js; host wiring belongs in bruno-electron.
4. **bruno-schema-types is types-only** (a *devDependency* of converters/filestore, built with
   plain `tsc`). Import types from it; never add runtime code or a runtime import of it.
5. **bruno-schema (Yup) and bruno-schema-types (TS types) are distinct and both live.** app +
   converters use `@usebruno/schema` for runtime validation; filestore + converters use
   `@usebruno/schema-types` for compile-time types. A data-model change usually touches both.

## Request execution pipeline

1. Renderer invokes `ipcMain.handle('send-http-request', …)` (`bruno-electron/src/ipc/network/index.js`).
2. Variables are interpolated via `interpolate` from `@usebruno/common`
   (`ipc/network/interpolate-vars.js`).
3. Pre-request scripts run via `ScriptRuntime` (`bruno-js/src/runtime/script-runtime.js`).
4. The request is built/sent — axios for HTTP, plus gRPC (`@grpc/grpc-js`) and WebSocket clients;
   shared building blocks (auth interceptors like `addDigestInterceptor`/`applyOAuth1ToRequest`,
   cookies, `scripting.buildScriptedEntry`, grpc/ws helpers) come from `@usebruno/requests`.
   bruno-requests is a *library of request pieces*, not the orchestrator — orchestration lives in
   `ipc/network`.
5. Post-response: **var extraction** via `VarsRuntime`, **tests** via `TestRuntime`, and
   **assertions** via `AssertRuntime` (all in `bruno-js/src/runtime/`) — these are distinct
   runtimes, not `ScriptRuntime`.
6. Result is returned to the renderer as a data object with an `error` field on failure (the
   handler does not reject — see `electron-ipc.md` error convention).

## Script sandboxing (`bruno-js`)

- **QuickJS** (default / "safe" mode): WebAssembly sandbox via `quickjs-emscripten`
  (`src/sandbox/quickjs/`).
- **Node VM** ("developer" mode): `node:vm` `runInContext` (`src/sandbox/node-vm/`).
- Runtime selection is centralized: `getJsSandboxRuntime` maps
  `collection.securityConfig.jsSandboxMode` (`'safe'` default → `SANDBOX.QUICKJS`; `'developer'`
  → `SANDBOX.NODEVM`) — see `bruno-js/src/utils/sandbox.js` and `ipc/network/index.js`. Route new
  sandbox-mode logic through `getJsSandboxRuntime`; do not re-derive the mapping.
- Scripts get a `bru`, `req`, `res` (post-response only), `test`, `expect` (chai), `assert`
  (chai), and `console` context (`script-runtime.js`).

## File format system

- `bruno-filestore` is the central parse/serialize entry (`src/index.ts`), delegating to
  `formats/bru` and `formats/yml`.
- Two on-disk formats: **`.bru`** (Bru DSL) and **`.yml`** (OpenCollection YAML). New collections
  default to **`.yml`** (`DEFAULT_COLLECTION_FORMAT = 'yml'`).
- The current `.bru` grammar is **v2 (ohm-js)** (`bruno-lang/v2/src/`, used by filestore via
  `bruToJsonV2`/`jsonToBruV2`). **v1 (arcsecond)** (`bruno-lang/v1/src/`) is legacy.
- Collections live on the filesystem and are watched by chokidar.

Any on-disk shape change → follow `.claude/rules/dsl-changes.md`.

## Redux store (app side)

The authoritative slice list is the `reducer` map in
`bruno-app/src/providers/ReduxStore/index.js` (~19 slices; `collections/` is by far the largest).
Conventions, middleware, and side-effect boundaries: `.claude/rules/redux-store.md`.

## Providers (`bruno-app/src/providers/`)

Read the directory for the current set: `App/`, `ReduxStore/`, `Theme/`, `Hotkeys/`, `Toaster/`,
`PromptVariables/`, plus paid-only `LicenseGuard/` (premium gating) and `Dictionary/` (i18n).

## Key types (`bruno-schema-types`)

Core types live in `packages/bruno-schema-types/src/` — read them for the current shapes. The
top-level request is a union: `Request = HttpRequest | GrpcRequest | WebSocketRequest`
(`requests/index.ts`); code that handles "a request" must account for all three. Other central
types: `Collection` (`collection/collection.ts`), `Item` (request or folder, discriminated by
`type`, `collection/item.ts`), `Auth` (a `mode` union, `common/auth.ts`), `KeyValue` (headers /
params / assertions, `common/key-value.ts`), `Environment`, `Script`.

## Key dependencies (actual majors — verify in the relevant `package.json` before assuming)

Several are pinned to majors below the latest — do **not** assume the newest API:

- Frontend (`bruno-app`): **react 19**, **@reduxjs/toolkit ^1.8 (v1, NOT v2)**,
  **react-redux ^7 (v7)**, **styled-components ^5 (NOT v6)**, **tailwindcss ^3**,
  **@rsbuild/core ^1.1**, **codemirror 5.65.2 (CodeMirror 5, NOT the scoped `@codemirror/*`)**.
- Desktop (`bruno-electron`): **electron ~37.6**, **electron-builder 24.13.3**, **chokidar ^3.5**,
  **@grpc/grpc-js ^1.13**, **js-yaml ^4.1**, **electron-store ^8.1**. (`ws` is a dep of
  bruno-requests/bruno-tests, **not** bruno-electron.)
- Parsing (`bruno-lang`): **arcsecond ^5** (v1, legacy), **ohm-js ^16.6** (v2, current).
  `bruno-toml` wraps **@iarna/toml** but is currently unused.
- **Hard pins in root `package.json` `overrides`: axios `1.16.0`, rollup `3.30.0`.** Bumping these
  in a leaf package has no effect — change the root override.
- **TypeScript is not uniform**: bruno-common `^5.8`, bruno-schema-types `^5.0`, but
  bruno-converters/filestore/graphql-docs/query/requests are `^4.8`. There is no root TS dep.
