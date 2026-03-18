# CondorDesk — Windows Desktop App Build Instructions

**Project:** `/home/jim/projects/optionstrader`
**Remote:** `git@github.com:sm911/optionstrader.git`
**Your git identity:** `herbie <herbie@securitymedic.com>` (already configured in the repo)

---

## Objective

Convert the CondorDesk trading dashboard from a web app into a standalone Windows `.exe` using **Electron**. The app should be a self-contained desktop application that runs locally with no external hosting.

---

## Current State

The project is a weekly options trading dashboard (iron condors, credit spreads) with:

- **Frontend:** React SPA (pre-built, in `/assets/` and `index.html`)
- **Backend:** Express/TypeScript server in `/server/`
- **Features:** Discord signal ingestion, TradingView webhook support, trade journal, risk management
- **Existing stack:** express, cors, helmet, express-rate-limit, winston, discord.js, zod

---

## Task

1. Package the existing React frontend + Express backend into a single installable `.exe`
2. Run the Express server as a background process inside the Electron main process
3. The React UI renders in the Electron BrowserWindow
4. On launch, show the existing splash screen, then the trading dashboard
5. Keep all existing functionality: Discord signal ingestion, TradingView webhooks, trade journal
6. Build output: a portable `.exe` and NSIS installer for Windows

---

## Target Architecture

```
CondorDesk.exe
├── Electron main process
│   ├── Embedded Express server (signals API, webhooks)
│   ├── Discord bot connection
│   └── Local data storage (JSON or SQLite)
└── Electron renderer (BrowserWindow)
    └── Existing React frontend
```

---

## Build Tooling

- Use `electron-builder` for packaging
- Target: Windows x64 (`--win --x64`)
- Output both portable `.exe` and NSIS installer
- Store user data in `%APPDATA%/CondorDesk/`

---

## Implementation Steps

1. Add `electron` and `electron-builder` as devDependencies to root `package.json`
2. Create `electron/main.ts` — launches Express server + BrowserWindow
3. Create `electron/preload.ts` — context bridge (if renderer needs Node APIs)
4. Move the Express server startup into a function that returns the running port
5. Point the BrowserWindow at `http://127.0.0.1:{port}`
6. Add `electron-builder` config to root `package.json`
7. Create build scripts: `npm run dev:electron` (dev) and `npm run build:win` (production)
8. Test on the local machine before pushing

---

## Target Repo Structure

```
optionstrader/
├── electron/
│   ├── main.ts          # Electron main process
│   └── preload.ts       # Context bridge
├── server/              # Existing Express backend (keep intact)
├── assets/              # Existing frontend assets (keep intact)
├── index.html           # Existing frontend entry (keep intact)
├── package.json         # Root package.json with electron + electron-builder config
├── .env.example         # Template for API keys
└── .gitignore           # Must exclude .env, dist/, *.exe, node_modules/
```

---

## Commit and Push Protocol

- After each meaningful milestone, commit and push:
  ```bash
  cd /home/jim/projects/optionstrader
  git add <specific files>
  git commit -m "descriptive message"
  git push origin main
  ```
- Use descriptive commit messages (e.g., "add Electron main process with embedded Express server")
- **ALWAYS run `git remote -v` before committing** to confirm you're in the right repo
- Never use `git add -A` or `git add .` — add specific files by name

---

## Security Rules (MANDATORY)

### 1. Never hardcode secrets
API keys (Discord bot token, TradingView webhook secret) must be loaded from:
- A `.env` file in the project root (for development)
- `%APPDATA%/CondorDesk/.env` (for production/packaged app)
- Never commit `.env` to git

### 2. Verify .gitignore before every commit
These must be in `.gitignore`:
```
.env
node_modules/
dist/
build/
release/
*.exe
*.msi
*.dmg
```

### 3. Electron security — no remote code execution
```typescript
// In electron/main.ts — ALWAYS use these settings:
const win = new BrowserWindow({
  webPreferences: {
    nodeIntegration: false,       // NEVER set to true
    contextIsolation: true,       // ALWAYS true
    sandbox: true,                // ALWAYS true
    preload: path.join(__dirname, 'preload.js'),
  },
});
```

### 4. Lock down the embedded server
- Bind Express to `127.0.0.1` only — never `0.0.0.0`
- The server must not be reachable from the network

### 5. Validate all inputs
- Keep existing TradingView webhook HMAC auth intact
- Keep existing Discord bot message parsing intact
- Never trust external data without validation

### 6. No shell.openExternal() with untrusted URLs
If opening external links from the renderer, validate the URL scheme (only allow `https://`).

### 7. Pin Electron version
Use a recent stable Electron version (v33.x or newer). Do not use outdated versions.

### 8. Content Security Policy
Set a restrictive CSP on the BrowserWindow:
```typescript
session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
  callback({
    responseHeaders: {
      ...details.responseHeaders,
      'Content-Security-Policy': ["default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; connect-src 'self' http://127.0.0.1:*"],
    },
  });
});
```

### 9. Never push build artifacts
The `.exe`, `dist/`, `build/`, and `release/` folders must never be committed to git.

### 10. Check three times before committing
Run `git status` and `git diff --staged` before every commit to ensure you're not committing secrets, build artifacts, or unintended changes.
