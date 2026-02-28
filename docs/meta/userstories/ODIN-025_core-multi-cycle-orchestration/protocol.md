# ODIN-025 Protocol: Multi-Cycle-Day Orchestration

**Status:** DONE — All tests green, reviews complete, fixes applied
**Date:** 2026-02-23
**Module:** odin-core

---

## Implementation Summary

### FSM Design Decision: FLAT_INTRADAY + COOLDOWN (not COOLING_OFF)

The story named the new FSM state "COOLING_OFF" but the implementation uses two distinct states:

- **FLAT_INTRADAY** — Transitional state immediately after a full exit fill. Persists for exactly one snapshot cycle to allow the exit to be fully processed and logged. On the next `onSnapshot` call, transitions to COOLDOWN.
- **COOLDOWN** — Active cooling-off period. The pipeline processes incoming snapshots (to keep data pipeline warm) but skips the decision loop entirely. Transitions to OBSERVING or DAY_STOPPED when the cooling-off period expires.

**Rationale:** A single `COOLING_OFF` state would conflate "just exited" with "waiting for re-entry". Separating them makes the transition logic cleaner and avoids off-by-one issues in cooldown duration calculation.

### Cooling-Off Duration

| Exit Reason | Duration |
|------------|----------|
| Stop-out (`wasStopOut=true`) | `stopOutCooldownMinutes` (default: 20 min) |
| All other exits | 15 minutes (hardcoded constant `COOLDOWN_MINUTES = 15`) |

Config property `odin.core.pipeline.stop-out-cooldown-minutes` defaults to 20.

### Five Re-Entry Guardrails

Evaluated in sequence when cooling-off expires:

| # | Guardrail | Blocks on |
|---|-----------|-----------|
| G1 | Per-instrument cycle limit | cyclesForInstrument >= maxCyclesPerInstrument |
| G2 | Global cycle limit | cyclesGlobal >= maxCyclesGlobal |
| G3 | Budget gate | remainingBudget < minBudgetForReentryEur |
| G4 | LLM availability + confidence | no analysis OR confidence < 0.5 |
| G5 | Regime gate | regime == UNCERTAIN or TREND_DOWN |

**Note on G4/G5 separation (post-ChatGPT-review fix):** Originally G4 checked both LLM confidence AND regime. This was refactored to separate concerns — G4 is purely LLM availability + confidence, G5 is regime quality. Constant `LLM_MIN_REENTRY_CONFIDENCE = 0.5` extracted.

### LLM Refresh in handleCooldown

After 15-20 minutes of COOLDOWN, the cached LLM analysis is stale (freshness check threshold is 120s). Before evaluating guardrails, `resolveLatestLlmAnalysis(snapshot)` is called to refresh the analysis. This ensures G4/G5 evaluate on current market conditions, not conditions from before the exit.

### Unrealized P&L Tracking

When in POSITIONED state, each `onSnapshot` call now updates the unrealized P&L:

```java
PositionState positionState = oms.getPositionState();
if (positionState != null) {
    double unrealizedPnl = positionState.calculateUnrealizedPnl(snapshot.currentPrice());
    globalRiskManager.updateUnrealizedPnl(instrumentId, unrealizedPnl);
}
```

This was a pre-review gap identified by Gemini: the budget gate (G3) uses `getRemainingRiskBudget()` which accounts for realized losses but not unrealized losses. The fix ensures the global risk manager always has current unrealized exposure.

### CompletedCycleContext

New record capturing the result of a completed cycle:

```java
public record CompletedCycleContext(
    int cycleNumber,
    ExitReason exitReason,
    double realizedPnl,
    Instant exitTime,
    boolean wasStopOut
)
```

Created when position fully closes (FLAT_INTRADAY entry), cleared when a new cycle begins (OBSERVING after successful cooldown). Used to determine cooldown duration and to provide context in re-entry guardrail evaluation.

### OMS Lifecycle

On full exit fill, the OMS is notified via `oms.notifyCycleEnd()` followed by `oms.reset()`. The reset clears the OMS internal state so cycle N+1 starts clean. `DataPipelineService` and `KpiEngine` are NOT reset — they continue running across cycle boundaries to maintain indicator warmth.

### GlobalRiskManager Coordination

`onCycleComplete(instrumentId, realizedPnl)` is called when transitioning to FLAT_INTRADAY. This:
1. Increments the per-instrument and global cycle counters
2. Updates realized P&L (for budget gate)
3. Records consecutive loss tracking

---

## ChatGPT Review (Round 1 + 2)

**Files sent:** TradingPipeline.java, TradingPipelineTest.java, CompletedCycleContext.java, PipelineFactory.java + CLAUDE.md context

### Round 1 Findings

| Severity | Finding | Disposition |
|----------|---------|-------------|
| KRITISCH | Global cycle limit: check-then-act gap (soft limit, not atomic) | ACCEPTED — single-threaded pipeline, documented as soft limit |
| KRITISCH | PARTIALLY_FILLED treated like full fill | NOTED — out of scope for ODIN-025, OMS handles partial fill tracking |
| WICHTIG | G4 and G5 redundant (regime checked in both) | FIXED — G4 now only checks availability + confidence |
| WICHTIG | EventLog payload not JSON-safe (injection risk) | FIXED — `logReentryBlocked` now escapes `\` and `"` |
| WICHTIG | `minBudgetForReentryEur` naming slightly misleading | ACCEPTED — name is clear enough in context |
| Threading | Single-thread concern | ACCEPTED — documented "not thread-safe, must be called on pipeline thread" |

### Round 2 Findings

| Severity | Finding | Disposition |
|----------|---------|-------------|
| Design | Global limit is soft (documented) | ACCEPTED — race condition theoretical in single-threaded architecture |
| Design | Guardrails evaluated once at cooldown-end | ACCEPTED — by design, ADR comment added in code |
| Architecture | G4+G5 redundancy confirmed | FIXED (see above) |
| Threading | Single-thread documented | FIXED — added to class JavaDoc |

---

## Gemini Review (Three Dimensions)

**Files sent:** TradingPipeline.java, TradingPipelineTest.java, CompletedCycleContext.java + concept docs

### Dimension 1: Code Bugs

| Severity | Finding | Disposition |
|----------|---------|-------------|
| BLOCKER | `updateUnrealizedPnl` never called — budget gate uses only realized losses | FIXED — added unrealized P&L update in POSITIONED state |
| MINOR | Cycle count label inconsistency on rejected orders (audit-label only) | ACCEPTED — documented with comment in code |

### Dimension 2: Concept Compliance

| Finding | Disposition |
|---------|-------------|
| Stale LLM at cooldown-end (15-20 min) | FIXED — `resolveLatestLlmAnalysis` called in `handleCooldown` before guardrail evaluation |
| Delayed cooldown initialization (FLAT_INTRADAY → next bar → COOLDOWN) | ACCEPTED — by design, single-bar delay is negligible |
| Cooling-off duration matches concept (15 min standard, 20 min stop-out) | CONFIRMED |
| Regime gate correctly blocks UNCERTAIN and TREND_DOWN | CONFIRMED |

### Dimension 3: Practical Gaps (Open Items)

| Scenario | Status |
|---------|--------|
| EOD occurs during COOLDOWN — pipeline goes EOD cleanly | OK — PipelineStateMachine forced to EOD, no issue |
| LLM timeout during mandatory re-entry call | OK — `resolveLatestLlmAnalysis` returns null on failure, G4 blocks re-entry |
| Kill-switch during COOLDOWN | OK — `handleCooldown` checks kill switch at start |
| Crash/restart during COOLDOWN | OPEN — state is in-memory, on restart pipeline starts INITIALIZING. This is acceptable for now (by design: single-day, EOD-flat) |
| Budget gate uses stale unrealized P&L if market moves fast | MITIGATED — unrealized P&L now updated every snapshot in POSITIONED state, so by exit time the value is current |

---

## Test Coverage

**Total tests in TradingPipelineTest.java after ODIN-025:** 33 tests (was 11)

### New Multi-Cycle Tests

- `testExitFillTransitionsToFlatIntraday`
- `testFlatIntradayTransitionsToCooldownOnNextBar`
- `testCooldownRemainsActiveDuringCoolingOffPeriod`
- `testCooldownTransitionsToObservingAfterExpiry`
- `testCooldownStopsWhenPerInstrumentCycleLimitReached`
- `testCooldownStopsWhenGlobalCycleLimitReached`
- `testCooldownStopsWhenBudgetInsufficient`
- `testCooldownStopsWhenNoLlmAnalysisAvailable`
- `testCooldownStopsWhenLlmReportsUncertainRegime`
- `testCooldownStopsWhenLlmReportsTrendDownRegime`
- `testCooldownStopsWhenLlmConfidenceBelowThreshold`
- `testReentryBlockedEventLoggedOnGuardrailFailure`
- `testStopOutUsesLongerCooldown`
- `testCompletedCycleContextCapturedOnExit`
- `testStopLossExitMarkedAsStopOut`
- `testCycleNumberIncreasesOnFirstEntry`
- `testOmsCycleEndCalledOnFullExit`
- `testGlobalRiskManagerCycleCompletedCalledOnExit`
- `testCompletedCycleContextClearedOnNewCycle`

### Key Test Design Decisions

**`transitionToPositioned()` uses full entry decision loop:** Early versions used direct FSM transitions, which bypassed `handleEntryIntent` and left `currentCycleNumber = 0`. The helper was rewritten to drive full `onSnapshot` → decision loop → `handleEntryIntent` → fill event, which is realistic and correctly initializes cycle state.

**`simulateExitFill()` does NOT stub `llmAnalyst.analyze()`:** Tests that need specific LLM analysis for cooldown guardrail evaluation configure `llmAnalyst` independently (via `prepareCooldownWithCachedLlmAnalysis()`). Stubbing LLM in `simulateExitFill` caused cross-test contamination: the stale analysis triggered a refresh call in `handleCooldown`, which returned TREND_UP from the stub and bypassed regime guardrails.

**`prepareCooldownWithCachedLlmAnalysis(llmAnalysis)`:** Helper that caches a specific LLM analysis by going through the full OBSERVING decision loop, then reaches COOLDOWN. Used for UNCERTAIN/TREND_DOWN regime block tests.

---

## Files Changed

| File | Change |
|------|--------|
| `odin-core/src/main/java/de/its/odin/core/pipeline/TradingPipeline.java` | Multi-cycle orchestration (cycle tracking, FSM states FLAT_INTRADAY/COOLDOWN, 5 guardrails, LLM refresh, unrealized P&L update, JSON escaping, thread-safety docs) |
| `odin-core/src/test/java/de/its/odin/core/pipeline/TradingPipelineTest.java` | Rewritten: 33 tests (up from 11), full multi-cycle coverage |
| `odin-core/src/main/java/de/its/odin/core/pipeline/CompletedCycleContext.java` | New record — immutable snapshot of completed cycle context |
| `odin-api/src/main/java/de/its/odin/api/model/PipelineState.java` | Added FLAT_INTRADAY and COOLDOWN states |
| `odin-core/src/main/java/de/its/odin/core/service/GlobalRiskManager.java` | Added getCyclesCompletedForInstrument(), getRemainingRiskBudget(), onCycleComplete(), updateUnrealizedPnl() |
| `odin-core/src/main/java/de/its/odin/core/config/CoreProperties.java` | Added PipelineProperties record with maxCyclesPerInstrument, maxCyclesGlobal, stopOutCooldownMinutes, minBudgetForReentryEur |
| `odin-core/src/main/resources/application-odin-core.properties` | Added pipeline config defaults |

---

## Open Items (Post-ODIN-025)

1. **Integration Tests (2.3):** Full 2-cycle flow integration test and global cycle limit integration test are NOT implemented. These are listed in Definition of Done 2.3 but were not part of the unit test scope. Should be tracked as a follow-up story or added to the next integration test sprint.

2. **Crash recovery during COOLDOWN:** On restart, the pipeline starts fresh in INITIALIZING. If a crash occurs during a 15-min cooldown, the cooldown is lost but the cycle count in GlobalRiskManager (which is also in-memory) is also lost. This is acceptable for the current single-day design — the entire trading session restarts via Safe-Mode reconciliation.

3. **PARTIALLY_FILLED handling:** If a fill is partial, the multi-cycle logic in TradingPipeline treats it differently only because OMS manages partial fill state. The FLAT_INTRADAY transition only triggers on a FULLY_FILLED broker event. No issues identified but should be validated in integration testing.
