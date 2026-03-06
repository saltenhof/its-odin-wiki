# ODIN-119 Protocol — Backtest-Live Store und Stream-Hook

## Status: COMPLETED

## Build & Test Results

| Check | Result |
|-------|--------|
| `tsc --noEmit` | PASS (no errors from ODIN-119 files; pre-existing errors in IntraDayChart.tsx from ODIN-117 WIP) |
| `npm test -- --run` | PASS — 520 tests, 33 test files, 0 failures |
| `npm run build` | Pre-existing build errors from uncommitted ODIN-117 panel changes (not related to ODIN-119) |

## Commit

- **Hash:** `a68aa57`
- **Message:** `feat(ODIN-119): add backtestLiveStore and useBacktestStream SSE hook`
- **Branch:** `main`
- **Pushed:** Yes

## Files Changed (6 new files, 1407 lines)

| File | Description |
|------|-------------|
| `src/shared/config/memoryLimits.ts` | Central MEMORY_LIMITS constant (DECISION_EVENTS_MAX=1000, CHART_BARS_MAX=2000, TRADE_EVENTS_MAX=500, DAY_SUMMARIES_MAX=500) |
| `src/shared/config/memoryLimits.test.ts` | 5 tests for MEMORY_LIMITS values and immutability |
| `src/domains/backtesting/stores/backtestLiveStore.ts` | Zustand store with all state fields (progress, day summaries, decision events, detail level, connection/backtest status) and FIFO-eviction actions |
| `src/domains/backtesting/stores/backtestLiveStore.test.ts` | 27 tests: initial state, updateProgress, addDaySummary, addDecisionEvent, FIFO eviction boundaries, setDetailLevel, setConnectionStatus, setBacktestStatus, reset |
| `src/domains/backtesting/hooks/useBacktestStream.ts` | Single-owner SSE hook with event routing, StrictMode handling, detailLevel reconnect, chart callback via useRef |
| `src/domains/backtesting/hooks/useBacktestStream.test.ts` | 20 tests: connection lifecycle, event routing to store, chart callback invocation, stream-gap/onReconnect, null payload edge cases, unknown event types, connectionStatus transitions |

## Test Coverage Summary

- **Store tests (27):** Initial state, all actions, FIFO eviction at exact boundary, eviction overflow, falsy-but-valid values (estimatedRemainingMs=0), backtestId stripping, reset from populated state
- **Hook tests (20):** Client creation, connect/disconnect lifecycle, unmount grace period, detailLevel reconnect, StrictMode double-mount, all 8 event types routed correctly, chart callback presence/absence, stream-gap with/without onReconnect, null riskRewardRatio/positionSize, unknown event type, connectionStatus connected-on-first-event
- **Config tests (5):** All 4 limit values + immutability

## LLM Sparring Summary

### ChatGPT (1 call)
**Focus:** Test edge cases review
**Key findings incorporated:**
1. `connectionStatus` 'connected' was unreachable — fixed by setting on first event received
2. Added tests for: stream-gap -> onReconnect callback, null riskRewardRatio/positionSize, unknown event type, connectionStatus transition on first event
3. Added boundary test: addDaySummary at DAY_SUMMARIES_MAX-1 -> MAX (no eviction)

### Gemini (3 calls)
**Dimension 1 — Code Quality:** No violations found. TypeScript strict compliance confirmed, readonly on all interfaces, TSDoc complete, no magic numbers.
**Dimension 2 — Konzepttreue:** 100% compliance with ODIN-119 spec. All state fields, actions, memory limits, event routing, and lifecycle behavior match the story.
**Dimension 3 — Praxis:** Test coverage rated "exceptionally thorough." Suggested zero-value edge case for estimatedRemainingMs — incorporated.

## Acceptance Criteria Checklist

- [x] backtestLiveStore exists as Zustand store with all defined state fields and actions
- [x] useBacktestStream hook creates exactly one SSE client per backtestId
- [x] Hook routes backtest-bar-progress to store.updateProgress()
- [x] Hook routes backtest-day-summary to store.addDaySummary()
- [x] Hook routes backtest-quant-score, backtest-gate-result, llm-update, trade-update to store.addDecisionEvent()
- [x] Hook calls optional onChartEvent callback for chart-relevant events
- [x] MEMORY_LIMITS constant exists in src/shared/config/memoryLimits.ts
- [x] Store actions enforce FIFO eviction at configured limits
- [x] SSE client closed on hook unmount
- [x] SSE client reconnected on detailLevel change (close + re-open)
- [x] tsc --noEmit compiles (ODIN-119 files error-free)
- [x] Unit tests for store actions (eviction) and hook lifecycle exist

## Design Decisions

1. **Hook signature changed to options object:** The linter auto-refactored the third parameter from `onChartEvent?` to `options?: UseBacktestStreamOptions` containing both `onChartEvent` and `onReconnect`. This is cleaner for extensibility.
2. **connectionStatus 'connected' set on first event:** The SseClient's onopen handler updates globalStore, not the backtest store. Setting 'connected' on first SSE event received provides accurate local status.
3. **BacktestChartSseEvent type:** Created as a discriminated subset of SseEvent (market-snapshot | trade-update) rather than using the existing ChartUpdate type, since ChartUpdate is for lightweight-charts series data, not SSE routing.
4. **backtestStatus not set by hook:** The SSE event union does not include completed/failed/cancelled lifecycle events (they are legacy backend event names). backtestStatus will be set by the consuming component via REST polling or a separate mechanism.
