# CondorDesk PROJECT-PLAN.md — Review Feedback

Overall the plan is solid — clear mission, good phase ordering, testing baked in from day 1. Here are gaps to address before building:

---

## 1. Data source for live market data — decide NOW, not in Phase 2

Phase 2 says "add market context ingestion" (spot price, IV, expected move) but doesn't specify WHERE that data comes from. This is the single hardest dependency in the entire project and drives architecture, cost, and licensing.

**Options to evaluate:**
- **Broker API (IBKR, Schwab/TD Ameritrade)** — free with account, real-time, includes options chains
- **Polygon.io** — paid, good REST + WebSocket, options data available
- **Tradier** — free sandbox, cheap production, good options chain API
- **CBOE LiveVol / delayed feeds** — options-specific but expensive
- **Yahoo Finance (unofficial)** — free but unreliable, no options chains

**Recommendation:** Research and pick the data source in Phase 1, even if you don't integrate it until Phase 2. The choice affects the data model, refresh architecture, and cost.

---

## 2. Define "realtime" — what latency is acceptable?

"Realtime" is undefined in the plan. For iron condor trading (not scalping), these targets are reasonable:
- **Underlying price:** <5 second delay
- **IV / greeks:** <30 second refresh
- **Options chain snapshots:** <60 second refresh
- **Insight scoring:** <2 seconds after new data arrives

This drives the architecture choice: polling vs WebSocket vs SSE push to the frontend.

---

## 3. Phase 3 needs options chain data — Phase 2 doesn't add it

Phase 3 builds candidate iron condors with delta, credit/width, liquidity scoring. But Phase 2 only mentions "underlying spot price, IV, expected move." Options chain data (strikes, bids/asks, greeks per strike, open interest) is a separate and larger data problem.

**Fix:** Add options chain ingestion explicitly to Phase 2, or create a Phase 2.5.

---

## 4. Add a persistence layer in Phase 1

The plan has no database. localStorage works for frontend state, but these need server-side storage:
- Trade journal / position history
- P/L tracking over time
- User risk rules and preferences
- Scoring history (for backtesting later)

**Recommendation:** Add SQLite (via `better-sqlite3`) in Phase 1. It's zero-config, file-based, perfect for a local desktop app. One file in `%APPDATA%/CondorDesk/condordesk.db`.

---

## 5. Missing: user settings/preferences

The plan doesn't address user configuration:
- Preferred underlyings (SPX, SPY, QQQ, IWM, etc.)
- Target delta for short strikes
- Risk tolerance / max position size
- Profit target % and stop loss multiplier
- DTE preferences

These are needed early because they parameterize the scoring engine. Add a settings model and UI to Phase 1.

---

## 6. Packaging smoke test — automate it

The postmortem documented three stacking bugs in the packaged app that dev-mode testing missed. The testing strategy mentions "packaging validation" but doesn't specify HOW.

**Recommendation:** Add an automated smoke test to CI:
```yaml
- name: Smoke test packaged app
  run: |
    ./release/win-unpacked/CondorDesk.exe &
    sleep 10
    curl -f http://127.0.0.1:8080/ || exit 1
    curl -f http://127.0.0.1:8080/api/health || exit 1
    taskkill /im CondorDesk.exe /f
```

This catches static-root bugs, middleware ordering issues, and server startup failures.

---

## 7. Add a versioning strategy

When you push updates, how does the version bump? This matters for:
- GitHub Releases (tagged versions trigger the release job)
- The NSIS installer (shows version to user)
- Auto-update support (if added later)

**Recommendation:** Use semver in `package.json`. Bump manually or use `npm version patch/minor/major` which auto-creates a git tag. CI already builds on `v*` tags.

---

## Summary of recommended changes to the plan

| Gap | Add to Phase |
|-----|-------------|
| Data source decision | Phase 1 (research) |
| Latency targets | Phase 1 (define) |
| Options chain ingestion | Phase 2 |
| SQLite persistence | Phase 1 |
| User settings model | Phase 1 |
| Packaging smoke test | Phase 1 (CI) |
| Versioning strategy | Phase 1 |
