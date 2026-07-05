---
paths:
  - "packages/bruno-app/**/*"
---

# React / Redux Architecture

## Redux Store (slices)

Slices live in `src/providers/ReduxStore/slices/` (one slice per domain) — read that directory
for the current set. Notable ones: `collections/` is the largest (collection tree, requests,
environments, runner, drafts; ~200 actions); `app` holds preferences / sidebar / task queue;
`workspaces`, `tabs`, `globalEnvironments`, `logs` follow. Paid-only slices also live here
(e.g. `git/`, `license`, `trial`). Open a slice's file for its actual state and actions.

## Custom Middleware (`middlewares/`)

Custom middleware lives in `src/providers/ReduxStore/middlewares/` — read the directory for the
current set. Examples of what they do: process the async task queue (e.g. open a request after a
file write), detect unsaved changes into drafts, autosave, debug logging (dev only).

## Key Actions (collections slice)

Async actions live in `slices/collections/actions.js` — read it for the current set. Frequently
used: `openCollectionEvent` (handles the `main:collection-opened` IPC), `sendRequest`,
`saveRequest`, `importCollection`, `pasteItem`, `deleteItem`.

## Sidebar Collection Visibility

Collections appear in sidebar ONLY if in active workspace (`Sidebar/Collections/index.js`):

```javascript
collections.filter((c) =>
  activeWorkspace.collections?.some((wc) => normalizePath(wc.path) === normalizePath(c.pathname))
);
```

Collapsed folders do NOT render children in the DOM (conditional rendering).

## Sidebar DOM Structure

`.collection-item-name` is on the **row wrapper** div for ALL items (folders, requests, JS files).
Children include the name span and `.menu-icon`. Use this class for Playwright locators.

## Providers (`src/providers/`)

Providers live in `src/providers/` — read the directory for the current set. Examples: `App/`
(IPC event listeners in `useIpcEvents.js`, app init), `ReduxStore/` (store config), `Theme/`
(styled-components theme), `Hotkeys/`, `Toaster/`, plus paid-only providers (`LicenseGuard/`
premium gating, `Dictionary/` i18n).
