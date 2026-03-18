# CondorDesk — Postmortem: Blank Window Bug

## What happened

The Windows .exe launched but showed a completely blank window. No crash, no error — just empty.

## Root cause: Static file serving ran before Electron set the env var

When esbuild bundles `electron/main.ts` and `server/src/index.ts` into a single `dist-electron/main.js`, all top-level module code runs immediately at load time.

In `server/src/index.ts`, the static file serving was set up at the **top level**:

```typescript
// This ran at module load time — BEFORE createMainWindow() set the env var
const staticRoot = process.env.CONDORDESK_STATIC_ROOT || path.join(__dirname, '..', '..');
app.use(express.static(staticRoot, { ... }));
```

The execution order was:

1. Bundle loads → server module executes top-level code
2. `CONDORDESK_STATIC_ROOT` is **not yet set** (Electron hasn't called `createMainWindow()`)
3. Falls back to `path.join(__dirname, '..', '..')`
4. In the bundle, `__dirname` = `dist-electron/` → `../..` = `/home/jim/projects/` (wrong!)
5. Express tries to serve `/home/jim/projects/index.html` — doesn't exist → blank page
6. Later, `createMainWindow()` sets `CONDORDESK_STATIC_ROOT` — **too late**, static middleware already mounted with the wrong path

## The fix (commit 3d10c11)

Moved the static file serving from **top-level module code** into the `startServer()` function. This way it runs **after** Electron has set `CONDORDESK_STATIC_ROOT`:

```typescript
// BEFORE (broken): top-level, runs at import time
const staticRoot = process.env.CONDORDESK_STATIC_ROOT || path.join(__dirname, '..', '..');
app.use(express.static(staticRoot, { ... }));

// AFTER (fixed): inside startServer(), runs after env var is set
export async function startServer(options) {
  const staticRoot = process.env.CONDORDESK_STATIC_ROOT || path.join(__dirname, '..', '..');
  app.use(express.static(staticRoot, { ... }));
  // ...
}
```

## Also fixed (commit 9caa6e2)

Added `asarUnpack` to `package.json` so `index.html` and `assets/` are extracted as real files on disk. Express's `express.static()` can't reliably serve files from inside an asar archive:

```json
"asarUnpack": [
  "assets/**/*",
  "index.html"
]
```

## Verified

- Tested on Linux with `npx electron . --no-sandbox`
- Server starts, reports correct static root
- `curl` confirms `index.html` and `assets/*.js` serve with HTTP 200
- Both fixes pushed to main

## Lesson for future Electron + Express bundling

**Never evaluate paths at module top level when bundling with esbuild.** Any path that depends on Electron APIs (`app.isPackaged`, `process.resourcesPath`) or env vars set by Electron must be deferred to a function that runs after Electron is ready.
