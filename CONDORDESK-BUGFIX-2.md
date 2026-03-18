# CondorDesk — URGENT Bugfix #2: App launches but is blank

**Project:** `/home/jim/projects/optionstrader`
**Priority:** Fix immediately

---

## Error

The Windows app opens but shows a completely blank/empty window. No crash, no error dialog — just empty.

## Root Cause

In `server/src/index.ts` line 59:
```typescript
const staticRoot = path.join(__dirname, '..', '..');
```

When bundled with esbuild into `dist-electron/main.js`, `__dirname` resolves to `dist-electron/` inside the asar. So `path.join(__dirname, '..', '..')` goes OUTSIDE the asar — the static files (`index.html`, `assets/`) can't be found.

## Fix

The static root needs to be calculated differently when running inside Electron vs standalone. The simplest fix:

In `server/src/index.ts`, replace:
```typescript
const staticRoot = path.join(__dirname, '..', '..');
```

With:
```typescript
const staticRoot = process.env.CONDORDESK_STATIC_ROOT || path.join(__dirname, '..', '..');
```

Then in `electron/main.ts`, before calling `startServer()`, set:
```typescript
// Point Express at the asar root where index.html and assets/ live
if (app.isPackaged) {
  process.env.CONDORDESK_STATIC_ROOT = path.join(process.resourcesPath, 'app.asar');
} else {
  process.env.CONDORDESK_STATIC_ROOT = path.join(__dirname, '..');
}
```

This way:
- **Packaged app:** static files are served from inside the asar
- **Development:** static files are served from the repo root
- **Standalone server:** falls back to the original `__dirname` logic

## Test

1. `npm run build && npm run start:electron` — verify the splash screen and dashboard render
2. `npm run build:win` — build the .exe
3. Run the .exe — verify it's not blank

## Commit and push

```bash
cd /home/jim/projects/optionstrader
git remote -v
git add server/src/index.ts electron/main.ts
git commit -m "Fix blank window — resolve static root for packaged Electron app"
git push origin main
```
