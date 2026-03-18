# CondorDesk — URGENT Bugfix: Windows .exe crashes on launch

**Project:** `/home/jim/projects/optionstrader`
**Priority:** Fix immediately

---

## Error

The Windows .exe crashes on launch with:
```
Error: Cannot find module '../server/dist/index'
```

## Root Cause

The `electron/main.ts` imports `from '../server/dist/index'` which compiles to a require of `../server/dist/index`. Inside the packaged asar, the path resolution breaks because:

1. `dist-electron/electron/main.js` tries to resolve `../../server/dist/index` — but the asar flattens the structure
2. `server/node_modules/` is NOT included in the `files` list in `package.json`, so even if the server code is found, its dependencies (express, cors, helmet, etc.) won't be

## Fix Required

Two things need to change:

### 1. Fix the import path in `electron/main.ts`

Change line 6 from:
```typescript
import { startServer, stopServer } from '../server/dist/index';
```
to:
```typescript
import { startServer, stopServer } from '../server/src/index';
```

Wait — that won't work either since we need compiled JS. The real fix is to make the compiled electron main.js resolve the server correctly inside the asar.

**Best approach:** Change the `tsconfig.electron.json` to compile both electron AND server code into a single output directory, OR use a bundler (like esbuild) to bundle the electron main process + server into a single file with all dependencies inlined.

### Recommended fix: Use esbuild to bundle the main process

```bash
npm install --save-dev esbuild
```

Add a script to `package.json`:
```json
"build:main": "esbuild electron/main.ts --bundle --platform=node --outfile=dist-electron/main.js --external:electron --format=cjs"
```

Update `package.json`:
```json
"main": "dist-electron/main.js"
```

Update `build.files` to:
```json
"files": [
  "dist-electron/**/*",
  "assets/**/*",
  "index.html",
  "package.json"
]
```

Update `build.extraMetadata`:
```json
"extraMetadata": {
  "main": "dist-electron/main.js"
}
```

This bundles the Express server, all its dependencies, and the Electron main process into a single JS file. No more missing module issues.

### 2. Update the CI workflow

The `build-win.yml` build step should use the new build command:
```yaml
- name: Build application
  run: npm run build
```

Make sure `npm run build` runs both `build:server` (if still needed) and `build:main`.

### 3. Test locally before pushing

```bash
npm run build
npm run start:electron
```

Verify the app launches, then:
```bash
npm run dist:win
```

Verify the .exe runs (even on Linux you can check the asar contents).

### 4. Commit and push

```bash
cd /home/jim/projects/optionstrader
git remote -v  # verify correct repo
git add -A  # only because this is a known set of build config changes
git commit -m "Fix packaged app missing server module — bundle with esbuild"
git push origin main
```

This will trigger a new CI build on the Windows runner.
