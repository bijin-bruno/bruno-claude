---
paths:
  - "packages/bruno-electron/**/*"
---

# Electron Main Process Guide

## IPC Startup Sequence

```
renderer mounts
  -> useEffect: ipcRenderer.invoke('renderer:ready')    [useIpcEvents.js]
     -> main: await onboardingPromise                    [preferences.js]
     -> main: send main:load-preferences
     -> main: send main:load-global-environments
     -> main: ipcMain.emit('main:renderer-ready')
        -> [async] workspace.js listener:
             await ensureDefaultWorkspaceExists()
             send main:workspace-opened (per workspace)
             ipcMain.emit('main:workspaces-ready')
                -> onboarding.js sends main:collection-opened

did-finish-load (independent, from Chromium):
  -> wraps webContents.send with JSON roundtrip          [index.js]
  -> send main:app-loaded  -->  sets data-app-state="loaded"
```

## IPC Race Condition Pattern

When `ipcMain.emit` triggers multiple `ipcMain.on` listeners:
- Sync listeners complete fully before the next starts
- Async listeners yield at first `await`, letting the next listener start
- `webContents.send` from earlier listeners arrives at renderer before messages from later async continuations

Fix: use chained events (e.g., `main:workspaces-ready`) to enforce ordering.

## IPC Handlers (`src/ipc/`)

IPC handlers live in `src/ipc/`, one file/dir per domain — read the directory for the current
set. Examples: `network/` (HTTP/GraphQL/gRPC/WebSocket execution), `collection.js`
(collection CRUD, import/export), `workspace.js` (workspace lifecycle), `preferences.js`
(preferences + onboarding orchestration), `filesystem.js` (dialogs), plus paid-only handlers
(e.g. `secrets.js` for external Vault/AWS/Azure secrets, `license.js`). Open a file to see the
channels it registers.

## Persistent Stores (`src/store/`, electron-store)

Stores are electron-store JSON files under `app.getPath('userData')`, defined in `src/store/` —
**read the directory for the current set**. Examples: `preferences.js` (also holds
lastOpenedCollections in the same `preferences.json`), `cookies.js`, `oauth2.js`,
`collection-security.js`.

## File Watchers (`src/app/`)

Watchers live in `src/app/` — **read the directory for the current set**. Examples:
`collection-watcher.js` (watches collection dirs, emits tree updates), `workspace-watcher.js`,
`gitWatcher.js`, `dotenv-watcher.js`.

All watchers must implement cleanup, called from `closeAllWatchers()` in `index.js` during
shutdown. When adding a new watcher, register its cleanup there too.

## Onboarding Flow

Entry: `src/app/onboarding.js`

1. Runs immediately at `registerPreferencesIpc` call
2. Checks `hasLaunchedBefore` from preferences
3. New users: imports sample collection from `resources/data/sample-collection.json`
4. Defers `main:collection-opened` until `main:workspaces-ready`
5. `DISABLE_SAMPLE_COLLECTION_IMPORT=true` skips import (default in Playwright)

## Key Environment Variables

- `ELECTRON_USER_DATA_PATH` — Custom userData path (dev mode only, index.js:22)
- `DISABLE_SAMPLE_COLLECTION_IMPORT` — Skip sample collection on first launch
- `PLAYWRIGHT` — Signals app is running under Playwright
- `DISABLE_SINGLE_INSTANCE` — Allow multiple app instances
