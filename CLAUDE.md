# Project Guide

Bruno fork — open-source API client (Electron + React + Redux).

Bruno is an API **client**, not a server: it sends requests and handles responses (auth, params, headers, bodies, environments, cookies). Judge every behavior and edge case by "what should a *client* do here?" — surface malformed or hostile responses clearly and stay robust; don't enforce server-side correctness.

## Quick Commands

```bash
npm run dev              # Run electron + react concurrently
npm run dev:watch        # Same, with hot-reload of the electron main process
npm run dev:web          # React dev server only (port 3000)
npm run dev:electron     # Electron only (needs dev:web running)
npm run dev:electron:debug  # Electron main with debugger attached
npm run storybook        # Component dev (bruno-app)
npm run test:e2e         # Playwright e2e (default + system-pac projects)
npm test --workspaces    # Unit tests (Jest) across all packages
npm run lint:fix         # ESLint fix
npm run build:electron   # Production build (add :mac/:win/:linux)
```

Setup: `nvm use && npm i --legacy-peer-deps && npm run setup` (Node **v22.12.0**, see `.nvmrc`).

### Running a single test

```bash
# Single unit test (Jest) — scope to the workspace, pass a path/pattern after --
npm test --workspace=packages/bruno-app -- path/to/file.spec.js
npm test --workspace=packages/bruno-requests -- -t "test name pattern"

# Single e2e test (Playwright) — a project flag is REQUIRED (config has no default)
npx playwright test tests/collection/create-collection.spec.ts --project=default
npx playwright test --project=default -g "test name pattern"
```

Other e2e projects: `test:e2e:ssl`, `test:e2e:auth`, `test:e2e:secrets-manager` (60s timeout).

### Building shared packages

`npm run dev` does **not** rebuild the shared packages (bruno-common, bruno-requests,
bruno-filestore, bruno-converters, bruno-query, bruno-schema-types, bruno-graphql-docs).
After editing one, rebuild it or run its watcher, or the app won't pick up changes:

```bash
npm run build:bruno-common      # also :bruno-requests, :bruno-filestore, etc.
npm run watch:common            # also watch:requests, watch:converters
```

## Monorepo Structure (npm workspaces)

Packages live under `packages/`, with top-level `tests/` and `playwright/`. Full package map
with per-package descriptions: `.claude/rules/architecture.md`.

## Key Architecture

- **Main process entry**: `packages/bruno-electron/src/index.js`
- **Renderer + Redux store**: `packages/bruno-app/src/providers/ReduxStore/`

Internals — request pipeline, IPC startup, stores, file watchers, slices/middleware, file
format, sandbox, core types, and dependency versions — live in the `.claude/rules/` files
indexed under **Detailed Rules** below.

## Testing

- **Unit**: Jest, config per-package (`packages/*/jest.config.js`); bruno-app uses jsdom + `jest.setup.js`.
- **E2E**: Playwright — fixtures, helpers, isolation rules, and pitfalls live in
  `.claude/rules/testing.md`; use the `write-e2e-test` skill when adding specs.

## Coding Standards

**Source of truth: `CODING_STANDARDS.md`** — read it for the full list. Highest-signal rules:

- 2 spaces, single quotes, semicolons; **no trailing commas**; always parenthesize arrow params `(x) =>`; JSX/TSX attrs use double quotes.
- React: avoid `useEffect` (prefer derived state / custom hooks); import hooks by name, never `React.useX`; a component is controlled XOR uncontrolled; Tailwind for layout, styled-components `theme` for colors.
- Extract for readability or clear reuse when it helps; avoid only unnecessary abstraction (indirection with no payoff). Add `data-testid` attributes for Playwright selectors.

## Detailed Rules (`.claude/rules/`)

Read these before non-trivial work in the relevant area:
`electron-ipc.md` (IPC handlers + startup sequence), `redux-store.md` (slices/middleware),
`testing.md` (e2e patterns & gotchas), `cross-platform.md` (Windows file/process/path pitfalls),
`dsl-changes.md` (on-disk `.bru`/`.yml` format & backward compatibility),
`conventions.md` (coding-standards & readability review lens),
`ai-hygiene.md` (comment & code cleanliness — no situational/obvious comments).
Architecture, core types, and dependency versions live in `architecture.md` (on-demand reference).
