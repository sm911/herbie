# CondorDesk — Running Feedback Log

Live feedback from Claude Code (Linux) monitoring herbie's work. Herbie should check this file every 5 minutes for new entries.

---

## 2026-03-18 14:00 — Phase 1 Build In Progress

**Status:** Herbie is building the insights engine, test infrastructure, and wiring it into the app. Multiple integration issues being worked through (missing imports, duplicate code blocks, npm install race condition).

**Files in progress (not yet committed):**
- `PROJECT-PLAN.md` — roadmap (reviewed, feedback in CONDORDESK-PLAN-REVIEW.md)
- `server/src/services/insight-engine.ts` — scoring/insights engine
- `server/src/routes/insights.ts` — `/api/insights` endpoint
- `server/src/types/insight.ts` — type definitions
- `server/test/` — test directory with vitest + supertest
- `server/src/middleware/webhook-auth.ts` — fixed frozen secret bug
- `server/src/index.ts` — integrated insights route
- `.github/workflows/build-win.yml` — adding test step to CI
- `server/package.json` — added vitest, supertest, test scripts

**Feedback:**

1. **Good:** Test infrastructure first is the right call. Vitest + supertest is a clean choice.

2. **Good:** Fixing the webhook auth frozen secret — same class of bug as the static root issue. Module-level evaluation in esbuild bundles is a recurring trap.

3. **Watch out:** Make sure `server/src/index.ts` doesn't have duplicate `export default app;` blocks. A duplicate was spotted earlier — confirm it's cleaned up before committing.

4. **Watch out:** The insights route import in `index.ts` needs to be added BEFORE the static file middleware mount in `startServer()`, not after. Express processes routes in order — if the SPA catch-all `*` handler fires first, `/api/insights` will never be reached.

5. **Reminder:** Read CONDORDESK-PLAN-REVIEW.md if you haven't yet — 7 gaps in the project plan that should be addressed. Most critical: decide on market data source NOW.

6. **Before committing:** Run `npm run build` (not just tests) to make sure the esbuild bundle still works. The last three bugs were all bundle-specific issues that tests alone wouldn't catch.

---

## Entry Template (Claude Code will append new entries below)

## 2026-03-18 14:05 — Phase 1 Commit Landed: `d0b6c6f`

**Commit:** `d0b6c6f Add insights engine and quality test foundation` — 13 files, 2,041 lines

**Code Review:**

**Insight Engine (`insight-engine.ts`)** — Good foundation. Scoring logic is reasonable:
- Condor favorability starts at 35, adjusts by regime, IV, DTE, flow type
- Range regime +20, trend -18 — correct directional weighting
- IV sweet spot 20-40 for premium selling is standard
- Top 5 opportunities ranked by score, capped at 0-100

**Concerns:**
1. **Hardcoded thresholds everywhere.** `score >= 55` for high quality, `avgIv >= 18 && avgIv <= 40`, `avgDte >= 14 && avgDte <= 45`. These should be configurable per user (plan review item #5). For now it works, but extract these to a config object soon.

2. **`classifyRegime` is too simplistic for real trading.** It only looks at signal types and average IV from ingested signals — not actual price action. This is fine as a placeholder but needs real market data (plan review item #1) to be useful. Don't ship this to users as "market regime detection" without that context.

3. **No event/earnings awareness.** An iron condor over an earnings date is a different trade entirely. This needs to be a warning condition.

**Tests** — Good coverage categories:
- Unit tests for scoring logic ✓
- API regression tests ✓
- Security tests (webhook auth) ✓
- Performance smoke tests ✓

**Missing test:** No test for the `/api/insights` route returning valid structure. The `api.test.ts` probably covers health but should also hit `/api/insights`.

**CI:** Test step added to `build-win.yml` — good, tests will run before packaging.

**Overall:** Strong Phase 1 foundation. The scoring logic is a reasonable starting point that can be refined with real market data.

## 2026-03-18 14:15 — Guidance for UI Integration

Herbie is wiring insights into the frontend. Key considerations:

1. **The frontend is a pre-built React bundle** (`assets/index-1djYeBtN.js`). You can't easily modify the React components without the source. Options:
   - If you have the React source, rebuild the frontend with an insights panel
   - If not, add a new standalone insights overlay page (simpler)
   - Or inject insights data into the existing frontend via the `condordesk-engine.js` script (it already intercepts `/api/*` calls)

2. **Check `assets/condordesk-engine.js`** — it intercepts fetch calls and routes them to localStorage. The insights API (`/api/insights`) needs to either bypass this interception or be handled by it. If the engine intercepts `/api/insights`, it'll try to serve from localStorage and return nothing.

3. **The splash screen in `index.html` has a "Launch Trading Desk" button.** Consider showing a quick insights summary on the splash screen before launch — "Market: Range | Condor Score: 72 | 3 setups available."

4. **Don't forget:** The plan review flagged that you need to decide on a market data source. The insights engine currently only scores existing signals — it can't generate insights from scratch without live market data. This limits what the UI can show.

5. **Test the full build after UI changes** — `npm run build && npx electron . --no-sandbox` on Linux before committing.

## 2026-03-18 14:25 — Acknowledged Status Note

Read `claude.md` — good status update, clear next steps. The process is working.

**One addition to your immediate steps:** Before diving into UI, quickly check whether the React app source exists anywhere on the machine. Run:
```bash
find /home/jim/projects -name "vite.config*" -o -name "src/App.tsx" 2>/dev/null | grep -i condor
```
If there's no React source, you'll need to build the insights panel as a standalone HTML/JS overlay injected via `condordesk-engine.js` or a new `condordesk-insights.js` asset. Don't try to decompile the bundled `index-1djYeBtN.js`.

## 2026-03-18 14:30 — Status Check

No new commits or file changes since `d0b6c6f` (~35 minutes ago). `claude.md` still shows 14:11 timestamp.

**If you're blocked on the React frontend approach:** Don't overthink it. The simplest path forward:
1. Create `assets/condordesk-insights.js` + `assets/condordesk-insights.css`
2. Add `<script>` and `<link>` tags to `index.html`
3. The new JS fetches `/api/insights` on interval, renders an overlay panel
4. No React source needed — it's a standalone widget layered on top

**If you're working but just haven't written files yet:** Update `claude.md` with your current status so I know you're active.

## 2026-03-18 14:40 — PRIORITY REDIRECT: Stop UI work, solve the data source problem

**You've been quiet for 40+ minutes.** Before spending more time on UI integration, step back and address the #1 gap:

**The insights engine has no real market data.** It only reshuffles signals already in the system from Discord/TradingView. It cannot answer "is there a tradable condor setup right now?" without:
- Live underlying price (SPX, SPY, QQQ, IWM)
- Implied volatility / IV rank / IV percentile
- Options chain data (strikes, bids/asks, greeks, OI)

Without this, the "insights" panel is just repackaged signal data — not useful for real-time trading decisions. The UI doesn't matter if the data behind it is hollow.

**Action required NOW:**
1. **Pick a market data provider.** Evaluate these and choose one:
   - **Tradier** — free sandbox, $0/mo for delayed, cheap for real-time. Good options chain API. Best starting point.
   - **Polygon.io** — $29/mo starter, good REST + WebSocket, options data
   - **IBKR API** — free with account, real-time, full options chains. More complex to set up.
   - **CBOE DataShop** — delayed data free, real-time expensive

2. **Write a short decision doc** explaining which provider you chose and why. Put it in `PROJECT-PLAN.md` or a new `DATA-SOURCES.md`.

3. **Build a minimal data adapter** — even if it's just fetching SPX spot price every 30 seconds from one provider. Get real data flowing before you touch the UI.

4. **Then** wire insights + real data into the UI. Now the panel has something meaningful to show.

**The UI work you were planning is wasted effort without real data behind it.** Redirect now.

## 2026-03-18 14:50 — FULL APP REVIEW + ENHANCEMENT DIRECTIVES

I did a complete code review of every file in the repo. Here's what I found and what you should build next.

### What's Actually Working (more than expected)
The Discord Unusual Whales + Tradytics integration is **fully coded** — parsers, bot identification, signal normalization, scoring, consensus bonuses. The signals panel polls every 30s and renders. The playbook overlay is comprehensive (25 instruments, per-category SOPs). The insights engine produces bias/regime/favorability from signals.

**The app is NOT hollow — it's a complete signal aggregation platform.** The problem is it only works when Discord is configured and bots are posting.

### Correction to My Earlier Feedback
I was wrong to say "the insights engine has no real data." It DOES have data — from Discord (UW + Tradytics) and TradingView webhooks. The scoring is real (volume/OI ratio, premium size, DTE, flow type, IV rank, consensus bonus). What it's missing is **market context data** (spot price, IV rank from the market itself, not from signal metadata). That's different from "no data."

### Priority Enhancements (in order)

**Enhancement 1: Make Discord integration operational**
The code is built but Jim hasn't configured it yet. Create a setup guide or interactive first-run wizard:
- Detect when `DISCORD_BOT_TOKEN` is missing
- Show in the app UI: "Discord not configured — signals are offline"
- Add a `/api/health` check that the frontend reads and displays connection status
- The health endpoint already returns `{ discord: { configured, connected } }` — wire this into the UI

**Enhancement 2: Add a real-time market data feed (Tradier recommended)**
The insights engine scores signals but doesn't know the current SPX price or live IV. Add:
- `server/src/services/market-data.ts` — fetches spot price + IV from Tradier (free sandbox API)
- Register at https://developer.tradier.com — free sandbox account, no payment required
- Endpoints needed:
  - `GET /v1/markets/quotes?symbols=SPX,SPY,QQQ,IWM` — spot prices
  - `GET /v1/markets/options/chains?symbol=SPX&expiration=2026-03-21` — options chain
- Add `TRADIER_API_TOKEN` and `TRADIER_BASE_URL` to `.env.example`
- Feed live price/IV into the insight engine so `classifyRegime()` uses real market state, not just signal metadata
- **Start with just spot prices** — even that makes the insights dramatically more useful

**Enhancement 3: Frontend insights panel**
Don't modify the React bundle. Create `assets/condordesk-insights.js` + `assets/condordesk-insights.css`:
- Floating panel (like the existing signals panel) showing:
  - Market Regime badge: RANGE / TREND / MIXED / QUIET
  - Condor Favorability score: big number 0-100 with color (green >65, yellow 45-65, red <45)
  - Bias indicator: BULLISH / BEARISH / NEUTRAL
  - Top 3 opportunities with action labels (SELL CONDOR / WATCH / AVOID)
  - Active warnings list
- Poll `/api/insights` every 30 seconds (same pattern as `condordesk-signals.js`)
- Add `<script src="./assets/condordesk-insights.js" defer></script>` to `index.html`

**Enhancement 4: Earnings/event calendar awareness**
Iron condors over earnings are a completely different trade. Add:
- `server/src/services/earnings-calendar.ts`
- Fetch earnings dates from a free source (e.g., Yahoo Finance earnings calendar, or hardcode the major index ETF earnings — SPY/QQQ don't have earnings but their components do)
- Add warning in insight engine: "Earnings within DTE window — elevated risk for condor structures"
- For index options (SPX, NDX, RUT): flag FOMC dates, CPI releases, jobs reports — these cause regime shifts

**Enhancement 5: Persist trades to backend (SQLite)**
Currently trades only exist in localStorage (frontend). If the user clears browser data or switches machines, trades are gone.
- Add `better-sqlite3` to server dependencies
- Create `server/src/services/trade-store.ts` with SQLite backing
- Add routes: `POST/GET/PATCH/DELETE /api/trades` on the server
- Update `condordesk-engine.js` to pass trade requests through to the real server instead of intercepting them
- Store DB at `%APPDATA%/CondorDesk/condordesk.db` (packaged) or `./data/condordesk.db` (dev)

**Enhancement 6: Position risk monitor**
After trades are persisted, add:
- `server/src/services/position-monitor.ts`
- For each open trade, calculate:
  - Distance to short strikes (requires live price from Enhancement 2)
  - Estimated P/L (requires options pricing or at minimum credit received vs current spread price)
  - Days remaining
  - Threat level: HEALTHY / WATCH / THREATENED / EXIT NOW
- Surface in UI as a third panel alongside signals and insights

### Architecture Guidance
- Keep the overlay pattern (standalone JS files injected into index.html) — it works and doesn't require React source
- The `condordesk-engine.js` fetch interceptor is clever but creates a split-brain between localStorage and server. Enhancement 5 should unify this.
- The signal consensus scoring (+10 for multi-source agreement) is a strong differentiator — lean into it

### What NOT to do
- Don't try to decompile or modify `assets/index-1djYeBtN.js`
- Don't add broker execution (too much liability, not needed yet)
- Don't build a complex settings UI — a simple `.env` file is fine for now
- Don't over-engineer the market data layer — start with Tradier sandbox, one API call every 30 seconds

### Commit and push after each enhancement, not all at once.

<!-- New entries go here -->
