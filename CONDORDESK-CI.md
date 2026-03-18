# CondorDesk — GitHub Actions CI for Windows Builds

**Project:** `/home/jim/projects/optionstrader`
**Remote:** `git@github.com:sm911/optionstrader.git`
**Priority:** Do this BEFORE any other CondorDesk work.

---

## Objective

Add a GitHub Actions workflow that builds the CondorDesk Windows `.exe` on a real Windows runner and uploads it as a GitHub Release asset. This replaces the local cross-compilation approach.

---

## Step 1: Create the workflow file

Create `.github/workflows/build-win.yml` in the optionstrader repo:

```yaml
name: Build Windows Installer

on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: windows-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install root dependencies
        run: npm ci

      - name: Install server dependencies
        run: npm ci --prefix server

      - name: Build server (TypeScript)
        run: npm run build:server

      - name: Build Electron main process
        run: npm run build:electron

      - name: Build Windows installer
        run: npx electron-builder --win --x64 portable nsis
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Upload portable exe as artifact
        uses: actions/upload-artifact@v4
        with:
          name: CondorDesk-portable
          path: release/*.exe
          retention-days: 30

      - name: Upload NSIS installer as artifact
        uses: actions/upload-artifact@v4
        with:
          name: CondorDesk-installer
          path: release/*.exe
          retention-days: 30

  release:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          path: artifacts

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: artifacts/**/*.exe
          generate_release_notes: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Step 2: Verify package-lock.json includes server deps

The workflow uses `npm ci` which requires `package-lock.json`. Make sure both the root and `server/` have lock files committed.

Check:
```bash
ls /home/jim/projects/optionstrader/package-lock.json
ls /home/jim/projects/optionstrader/server/package-lock.json
```

If `server/package-lock.json` is missing, run:
```bash
cd /home/jim/projects/optionstrader/server && npm install
```
and commit the lock file.

---

## Step 3: Commit and push

```bash
cd /home/jim/projects/optionstrader
git add .github/workflows/build-win.yml
git commit -m "Add GitHub Actions workflow for Windows builds"
git push origin main
```

This will immediately trigger the first build on push to main.

---

## Step 4: Verify the build

After pushing, the build will run automatically. Check status:
- Go to https://github.com/sm911/optionstrader/actions
- The "Build Windows Installer" workflow should be running
- It takes ~10 minutes on the Windows runner
- When done, the `.exe` files appear as downloadable artifacts on the workflow run

---

## Step 5: Create a tagged release (when ready)

To create a proper GitHub Release with the `.exe` attached:

```bash
cd /home/jim/projects/optionstrader
git tag v1.0.0
git push origin v1.0.0
```

This triggers the build AND the release job. The `.exe` will appear at:
https://github.com/sm911/optionstrader/releases/tag/v1.0.0

---

## How it works going forward

- **Every push to main:** builds the `.exe` and uploads as a workflow artifact (downloadable for 30 days)
- **Every version tag (`v*`):** builds the `.exe` AND creates a GitHub Release with the installer attached
- **Manual trigger:** click "Run workflow" on the Actions page anytime

---

## Security notes

- The workflow uses `GITHUB_TOKEN` (auto-provided by GitHub Actions) — no extra secrets needed
- No `.env` or API keys are used during the build — those are runtime-only
- Build artifacts are the `.exe` files only — no source code or secrets are exposed
- The `permissions: contents: write` is required for creating releases

---

## Commit protocol

- Commit the workflow file with a clear message
- Push to main to trigger the first build
- Monitor the Actions tab for success/failure
- If the build fails, read the logs, fix the issue, commit, and push again
- **ALWAYS run `git remote -v` before committing**
