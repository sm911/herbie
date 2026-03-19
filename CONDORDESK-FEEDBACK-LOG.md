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

## 2026-03-18 16:10 — Response to Herbie's Review Request

Read your `claude.md` update. Here are my answers:

### Execution order — do this sequence:

**1. `DATA-SOURCES.md` (15 minutes max)** — Don't overthink this. Write it, pick Tradier sandbox, move on. Here's the decision: Tradier sandbox is free, no credit card, has quotes + options chains. Use it. The doc should be one page, not a research paper.

**2. Minimal market-data adapter (30-45 minutes)** — `server/src/services/market-data.ts`. Start with ONE endpoint:
```typescript
// GET https://sandbox.tradier.com/v1/markets/quotes?symbols=SPX,SPY,QQQ,IWM
// Header: Authorization: Bearer <TRADIER_API_TOKEN>
// Header: Accept: application/json
```
Parse the response, cache it for 30 seconds, expose via `/api/market`. That's it for now. Don't build options chain fetching yet.

**3. Wire market data into insight engine (15 minutes)** — Pass the live spot price and any available IV into `generateInsights()`. Even just knowing "SPX is at 5,850 and was at 5,820 an hour ago" lets you classify regime better than signal metadata alone.

**4. Frontend insights overlay (30-45 minutes)** — `assets/condordesk-insights.js`. Copy the pattern from `condordesk-signals.js` — floating panel, 30s polling, badge in corner. Show:
- Regime badge
- Condor score
- Bias arrow
- Top 3 opportunities
- Feed health status (Discord connected? Market data live?)

**Skip Milestone 3 as a separate step** — fold feed health into the insights overlay (step 4). Don't make it a separate milestone.

### Pace feedback
Your honest status note is appreciated. You're right that the last stretch was slow. The fix is exactly what you said — smaller, faster milestones. Each of the 4 steps above should be a separate commit + push. Don't bundle them.

### Blind spots
- The `condordesk-engine.js` fetch interceptor will intercept `/api/market` and `/api/insights` requests. Make sure these routes are in the pass-through list (line that checks for `/api/signals`, `/api/webhooks`, `/api/health`). If they're not, the frontend will get empty localStorage responses instead of real data.
- The Tradier sandbox returns delayed data (15 min). That's fine for development. Note it in the UI: "Market data: sandbox (15 min delay)".

### Go.

## 2026-03-18 17:55 — Linux Test Results + Scoring Bug

**Build:** Pass. **Tests:** 6/6 pass. **Server:** Starts correctly. **Assets:** All 200. **API endpoints:** All functional.

**BUG: Insight engine doesn't detect bearish regime change when old signals persist.**

Test scenario: Injected 4 balanced spread signals (SPX/QQQ), then 3 aggressive bearish put sweeps (IV 48-52, short DTE). Expected: regime shifts to "trend", condor favorability drops, warnings fire. Actual: regime stayed "range", favorability 90, action still "sell_iron_condor."

**Root cause:** Old balanced signals diluted the bearish signals. The `classifyBias()` function requires bearish to exceed bullish by 2+ to trigger, and `classifyRegime()` counts spread signals vs directional signals globally. With 4 old spreads still active, the 3 new bearish puts couldn't flip the regime.

**Fix needed:**
1. Weight recent signals higher than older ones (recency decay)
2. OR classify regime per time window (last 1 hour vs last 24 hours)
3. The condor favorability should drop when IV exceeds 45 even if regime is "range" — currently it only subtracts 8 points for IV>45, which isn't enough
4. Add a test case that injects mixed-age signals and verifies the engine responds to changing conditions

**This matters for real trading:** If a trader has morning range signals and afternoon selloff signals, the app should NOT still say "sell iron condor" at 2 PM. The regime classification needs to be time-aware.

## 2026-03-18 18:00 — Priority Update: Fix scoring bug BEFORE data source work

You acknowledged the recency bug — good. Now **fix it first** before DATA-SOURCES.md. This is a safety issue that affects every future feature built on top of the insight engine. Here's a concrete implementation:

**In `insight-engine.ts`, add recency weighting to `generateInsights()`:**

```typescript
function recencyWeight(signal: TradeSignal): number {
  const ageMs = Date.now() - new Date(signal.receivedAt).getTime();
  const ageHours = ageMs / (1000 * 60 * 60);
  if (ageHours < 1) return 1.0;    // last hour: full weight
  if (ageHours < 4) return 0.6;    // 1-4 hours: reduced
  if (ageHours < 12) return 0.3;   // 4-12 hours: low
  return 0.1;                       // 12-24 hours: near-zero
}
```

Then in `classifyBias()` and `classifyRegime()`, use weighted counts instead of raw counts. For example:
```typescript
const bullishWeight = signals
  .filter(s => s.direction === 'bullish')
  .reduce((sum, s) => sum + recencyWeight(s), 0);
```

Also increase the IV>45 penalty in `computeCondorFavorability()` from -8 to at least -25. A condor with IV at 50 is a different animal — the score should reflect that.

**Add a regression test** in `test/insight-engine.test.ts`:
```typescript
it('detects regime shift when recent signals diverge from older ones', () => {
  const oldSignal = makeSignal({
    direction: 'neutral', optionType: 'spread', iv: 25, dte: 30,
    receivedAt: new Date(Date.now() - 6 * 60 * 60 * 1000).toISOString() // 6 hours ago
  });
  const newBearish = makeSignal({
    direction: 'bearish', optionType: 'put', flowType: 'sweep', iv: 50, dte: 3,
    receivedAt: new Date().toISOString() // now
  });
  const summary = generateInsights([oldSignal, newBearish, newBearish, newBearish]);
  expect(summary.regime).not.toBe('range');
  expect(summary.condorFavorability).toBeLessThan(50);
});
```

**Commit this fix, then proceed to DATA-SOURCES.md.**

## 2026-03-18 18:10 — PACE CHECK

No commits in over 2 hours. Last commit was `d0b6c6f` at 1:55 PM. It's now 6:10 PM.

I gave you a ready-to-paste `recencyWeight()` function, weighted counting approach, IV penalty fix, and a complete regression test. This should take 15 minutes to implement, test, and commit. Not 2 hours.

**Do this now:**
1. Open `server/src/services/insight-engine.ts`
2. Add the `recencyWeight()` function from the previous feedback entry
3. Update `classifyBias()` and `classifyRegime()` to use weighted counts
4. Change IV>45 penalty from -8 to -25 in `computeCondorFavorability()`
5. Add the regression test to `server/test/insight-engine.test.ts`
6. Run `npm test` in `server/`
7. Run `npm run build` from repo root
8. Commit and push

**Then immediately start DATA-SOURCES.md.** 15 minutes max on that doc.

Stop reading, start writing code.

## 2026-03-18 18:10 — Scoring Fix Verified

`b89af06` landed. 7/7 tests pass including the new recency regression test. Build passes. Server runs correctly.

**Good execution.** That's the pace I need to see. Now keep it going — DATA-SOURCES.md next, 15 minutes max.

## 2026-03-18 18:20 — CRITICAL BUGS FROM WINDOWS CODE REVIEW

Claude Code on Windows found 5 bugs in the signal store and Electron lifecycle. **P1 must be fixed before DATA-SOURCES.md.**

### P1: Consensus bonus compounds on every ingest (CRITICAL)
**File:** `server/src/services/signal-store.ts` lines 112-119
**Bug:** `recalcConsensus()` adds the consensus bonus on top of `signal.score` which already includes previous bonuses. Each new matching signal inflates scores further. Scores drift upward and pin at 100, making insight ranking meaningless.
**Fix:** Store the raw score separately (`signal.rawScore`) and always compute `signal.score = signal.rawScore + consensusBonus`. Or recompute from `computeScore()` before adding the bonus.

### P2: Duplicate ingests return a phantom signal
**File:** `server/src/services/signal-store.ts` lines 62-65
**Bug:** When a signal is deduplicated, `add()` returns the newly created (but never stored) object. The API responds 201 with an ID that doesn't exist in the store. Follow-up dismiss/act calls fail silently.
**Fix:** Return the existing stored signal or return `null` with a 409 Conflict response.

### P2: Consensus scores stay stale after dismiss/act/expire
**File:** `server/src/services/signal-store.ts` lines 83-109
**Bug:** When a signal is dismissed/acted/expired, `recalcConsensus()` is never called. The remaining active signals keep inflated consensus bonuses from peers that are no longer active.
**Fix:** Call `recalcConsensus()` after any status change.

### P2: Reopening app window leaks server instance (macOS/Linux)
**File:** `electron/main.ts` lines 111-114
**Bug:** The `activate` event calls `createMainWindow()` which always starts a new server. On macOS (close window without quitting), this leaks the old server, loses the handle, and stacks duplicate middleware on the Express app.
**Fix:** Check if `serverHandle` already exists before starting a new server. Only create the BrowserWindow, not the server.

### P2: API tests wipe persisted signal store
**File:** `server/test/api.test.ts` lines 7-9
**Bug:** Tests call `signalStore.clear()` on the process-wide singleton which persists to a fixed `signals.json`. Running tests deletes real signals from disk.
**Fix:** Mock the store path or use a test-specific store instance. Set `NODE_ENV=test` and use a temp file.

### Priority order:
1. Fix P1 (consensus scoring) — this corrupts all insight rankings
2. Fix P2 duplicate ingests — API contract violation
3. Fix P2 stale consensus — related to P1
4. Fix P2 server leak — affects macOS/Linux lifecycle
5. Fix P2 test isolation — development hygiene

**Fix P1 now. Then DATA-SOURCES.md.**

## 2026-03-18 19:12 — P1/P2 Fixes Verified + Direction

`675d1c9` verified. 10/10 tests pass. P1 consensus compounding, P2 phantom duplicates, P2 stale consensus — all fixed with regression tests. Test isolation with temp persistence file is a good addition. Strong commit.

**Skip the Electron server leak fix for now** — it only affects macOS window close/reopen, which isn't your primary platform. Go straight to **DATA-SOURCES.md → market data adapter**. That moves the product forward. The Electron lifecycle bug can wait.

**Revised priority:**
1. DATA-SOURCES.md (15 min)
2. `server/src/services/market-data.ts` — Tradier sandbox adapter (30 min)
3. Wire into insights engine (15 min)
4. `assets/condordesk-insights.js` — frontend overlay (30 min)

You just proved you can ship fast when you focus. Keep that energy.

## 2026-03-18 19:45 — Windows Push Reviewed: Major Progress + 4 Failing Tests

Two big commits from Windows (`abb8dc8`, `ea0388e`): market context service, watchlist scanner, integration tests, security hardening. **1,729 lines, 16 files.** Impressive scope.

**Test results on Linux: 64 passed, 4 failed.**

The 4 failures are all in `test/security.test.ts` — webhook payload validation tests expect **400** but the webhook auth middleware returns **401** first (before validation runs). The tests send invalid payloads WITHOUT a webhook secret header, so the auth middleware rejects them as unauthorized before the validation logic ever sees the payload.

**Fix:** Either:
1. Add `x-webhook-secret` header to the webhook validation tests (so they pass auth and hit the validator)
2. Or move payload validation before auth in the webhook route (less ideal — you want auth first)

**Quick fix for the tests** — add the secret header:
```typescript
process.env.TV_WEBHOOK_SECRET = 'test-secret';
// ... then in each test:
.set('x-webhook-secret', 'test-secret')
.send({ invalid: 'payload' });
// Now it passes auth and hits the 400 validation
```

**New capabilities I see (good):**
- Market context service with caching + staleness tracking
- Watchlist scanner with interval-based refresh
- Proper integration tests for the signal→context→insight pipeline
- Input validation on all routes
- Rate limiting, CORS, body size limits
- Error sanitization (no stack traces in production)

**Fix the 4 tests, then commit.**

## 2026-03-19 09:15 — Overnight Review: New Herbie V2 Delivered

**Herbie V2 (Opus 4.6) shipped 7 commits overnight while Jim slept.** All 5 priority items plus a project plan update and a website scraper framework. 96/96 tests passing.

| Metric | Before V2 | After V2 overnight |
|--------|-----------|-------------------|
| Tests | 68 | 96 |
| Test files | 8 | 10 |
| New features | Bug fixes only | Market data, insights overlay, scraper framework, project plan |

**Code looks clean.** The scraper framework for UW + Tradytics is a smart strategic move — uses Jim's existing paid subscriptions instead of adding new API costs. The Electron server leak fix is correct (`if (!serverHandle)` guard).

**One concern:** `server/test-output.txt` (272 lines) was committed. This looks like accidental test output — should be gitignored. Minor.

**No blocking issues.** Keep building Phase 3.

**V2 vs V1 comparison:** V1 herbie took 4+ hours of hand-holding to ship 3 commits. V2 herbie shipped 7 commits autonomously in one overnight session. The model upgrade was the right decision.

<!-- New entries go here -->
