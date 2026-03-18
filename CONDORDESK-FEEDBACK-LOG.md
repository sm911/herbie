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

<!-- New entries go here -->
