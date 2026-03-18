# CondorDesk — Postmortem: Blank Window Bugs

## Summary

The Windows .exe launched but showed a blank window. This took multiple rounds to fix because there were **three separate bugs** stacking on top of each other.

---

## Bug 1: Static root evaluated at module load time (commit 3d10c11)

### What happened
`server/src/index.ts` had static file serving at the **top level**:
```typescript
const staticRoot = process.env.CONDORDESK_STATIC_ROOT || path.join(__dirname, '..', '..');
app.use(express.static(staticRoot, { ... }));
```

When esbuild bundles everything into `dist-electron/main.js`, all top-level code runs immediately. `CONDORDESK_STATIC_ROOT` wasn't set yet (Electron hadn't called `createMainWindow()`), so it fell back to `path.join(__dirname, '..', '..')` which resolved to the wrong directory.

### Fix
Moved static file serving into `startServer()` so it runs after the env var is set.

### Lesson
**Never evaluate paths at module top level when bundling with esbuild.** Any path depending on Electron env vars must be deferred to a function.

---

## Bug 2: Error handler mounted before static routes (commit fb1a4a7)

### What happened
After moving static serving into `startServer()`, the error handler was still at the top level:
```typescript
app.use(errorHandler);  // mounted at load time

export async function startServer() {
  app.use(express.static(...));  // mounted later
}
```

Express processes middleware **in order of mounting**. The error handler caught everything before static files got a chance to serve.

### Fix
Moved `app.use(errorHandler)` into `startServer()`, after the static routes.

### Lesson
**Express middleware order matters.** Error handlers must be mounted LAST, after all routes and static serving.

---

## Bug 3: Server starting twice — the esbuild bundling trap (commit 68cd771)

### What happened
The server had a standalone guard:
```typescript
if (require.main === module) {
  void startServer({ host: '127.0.0.1', port: config.port });
}
```

When esbuild bundles `electron/main.ts` + `server/src/index.ts` into a single file, `require.main === module` is **always true** because both refer to the same top-level module (`dist-electron/main.js`).

This caused:
1. Bundle loads → guard fires → server auto-starts on port 8080 with WRONG static root
2. The auto-started server registers a `*` catch-all handler
3. Later, `createMainWindow()` sets correct env var and calls `startServer()` on a random port
4. New static middleware is appended to the SAME Express app, but the first `*` handler already catches everything
5. Every request → first catch-all → `res.sendFile('resources/index.html')` → ENOENT → blank

### Fix
Two changes:
```typescript
// 1. Prevent auto-start in Electron
if (require.main === module && !process.versions.electron) {
  void startServer(...);
}

// 2. Use app.getAppPath() instead of hardcoded paths
if (app.isPackaged) {
  process.env.CONDORDESK_STATIC_ROOT = app.getAppPath();
}
```

`app.getAppPath()` works regardless of whether asar is enabled or disabled.

### Lesson
**esbuild flattens everything into one module.** Guards like `require.main === module` become meaningless. Always add an explicit check for the runtime environment (e.g., `!process.versions.electron`).

---

## Additional fix: asar disabled (commit 2f1f000)

Express's `express.static()` can't reliably serve files from inside an asar archive. Disabled asar packaging so static files exist as real files on disk. This is fine for a personal tool.

---

## Key takeaways for future Electron + Express + esbuild projects

1. **Defer all path resolution** to runtime functions, never module top-level
2. **Express middleware order is critical** — static routes before error handler, always
3. **esbuild makes `require.main === module` useless** — use `process.versions.electron` to detect runtime
4. **`app.getAppPath()`** is the reliable way to find your app root in packaged Electron
5. **Disable asar** when using Express static serving (or use a different serving strategy)
6. **Always test the packaged app**, not just dev mode — they have different path resolution
7. **Test on Linux first** before shipping to Windows — faster iteration cycle
