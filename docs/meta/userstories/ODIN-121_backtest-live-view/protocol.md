# ODIN-121 Protocol — BacktestLiveView Page-Integration

## Runde: R1 (Erstimplementierung)
## Datum: 2026-03-06
## Commit: 82c72ef
## Branch: main

---

## Deliverables

| # | Deliverable | Status |
|---|-----------|--------|
| 1 | `src/domains/backtesting/pages/BacktestLiveView.tsx` + `.module.css` | Done |
| 2 | `src/domains/backtesting/components/DetailLevelToggle.tsx` + `.module.css` | Done |
| 3 | Route registration in BacktestingPage.tsx (`/:backtestId/live`) | Done |
| 4 | Unit tests: BacktestLiveView.test.tsx (13 tests), DetailLevelToggle.test.tsx (9 tests) | Done |
| 5 | protocol.md | Done |
| 6 | Build GREEN, all tests pass | Done |

## Build & Test

- **tsc --noEmit**: Clean, zero errors
- **npm run build**: Success (2.60s)
- **npm test -- --run**: 42 test files, 626 tests passed, 0 failures

## Files Changed (7 files, +939 lines)

| File | Action |
|------|--------|
| `src/domains/backtesting/pages/BacktestLiveView.tsx` | NEW |
| `src/domains/backtesting/pages/BacktestLiveView.module.css` | NEW |
| `src/domains/backtesting/pages/BacktestLiveView.test.tsx` | NEW |
| `src/domains/backtesting/components/DetailLevelToggle.tsx` | NEW |
| `src/domains/backtesting/components/DetailLevelToggle.module.css` | NEW |
| `src/domains/backtesting/components/DetailLevelToggle.test.tsx` | NEW |
| `src/domains/backtesting/BacktestingPage.tsx` | MODIFIED (route added) |

## Architecture Decisions

1. **Single-Owner SSE Pattern**: `useBacktestStream` is the sole SSE client. Chart receives events imperatively via `onChartEvent` ref-callback. Store receives events declaratively via Zustand actions.

2. **useRef for handleChartEvent**: Prevents stale closures by storing the chart event handler in a ref that is updated every render. A stable `useCallback` wrapper delegates to the ref.

3. **Isolated ProgressStrip**: Extracted progress/summary selectors into a dedicated component to prevent parent `BacktestLiveView` from re-rendering on every high-frequency bar-progress event (Gemini D3 recommendation).

4. **Store Reset on backtestId Change**: Added `useEffect` that calls `resetStore()` when backtestId changes or on unmount, preventing stale state bleed between backtest runs (ChatGPT recommendation).

5. **NaN Timestamp Guard**: Added `Number.isNaN()` checks before forwarding timestamps to chart imperative API (ChatGPT recommendation).

6. **MarketSnapshot as Tick Data**: MarketSnapshotPayload carries a single price (not OHLCV), so we use `updateActiveBar` instead of `appendBar` to fold ticks into the active bar.

## LLM Sparring Summary

### ChatGPT (1 Call)
- **Focus**: Test edge cases, stale closure risks, reconnect races, memory leaks
- **Key Findings**:
  - Store must be reset on backtestId change (IMPLEMENTED)
  - NaN timestamp guard needed for chart methods (IMPLEMENTED)
  - Connection-generation gating would further protect against stale SSE events (NOTED for future)
  - Tests need more reconnect-race coverage (NOTED for future)
- **Rating**: Architecture sound, needs generation gating for full reconnect safety

### Gemini D1 — Code Quality (Rating: 9/10)
- TypeScript strict compliance excellent
- Hook patterns (useRef for stale closure avoidance) rated as "textbook perfect"
- No `any` types, all readonly modifiers present
- Minor: suggested clsx utility for class concatenation

### Gemini D2 — Concept Compliance (Rating: 9.5/10)
- All 8 concept requirements verified as compliant
- Noted route path is `/backtesting/:backtestId/live` vs spec `/backtests/:backtestId/live` — domain router mounts under `/backtesting/*` per existing convention

### Gemini D3 — Practical Production Readiness (Rating: 8.5/10)
- Memory management for 30-90min runs: excellent (FIFO eviction)
- Performance concern: volatile completedBars selector caused parent re-renders (FIXED: extracted ProgressStrip)
- CSS layout robust for desktop
- Error handling and UX rated as practical and intuitive

## Deferred Items (Not In Scope)
- Connection-generation gating for SSE reconnect races (ODIN-123 territory)
- Reconnect-race integration tests with fake SSE client
- Chart event buffering before chart mount
