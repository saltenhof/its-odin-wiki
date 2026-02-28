# ODIN-027 Implementation Protocol — EOD Forced Close Timing Logic

**Story:** ODIN-027
**Status:** Implemented, reviewed, post-review fixes applied, all tests green
**Date:** 2026-02-23
**Implementor:** Claude Sub-Agent (claude-sonnet-4-6)

---

## 1. Summary

Implemented the four-phase EOD forced close sequence for all active trading pipelines in `odin-core`.

### Files Created
| File | Description |
|------|-------------|
| `odin-core/.../service/EodPhase.java` | New enum: PHASE_1..PHASE_4 |
| `odin-core/.../service/EodSequencer.java` | New stateful POJO service for 4-phase EOD sequence |
| `odin-core/.../service/EodSequencerTest.java` | 23 unit tests |
| `odin-core/.../service/EodSequencerIntegrationTest.java` | 4 integration tests (real SimClock) |

### Files Modified
| File | Change |
|------|--------|
| `CoreProperties.java` | Added `EodProperties` nested record with `@AssertTrue` cross-field validation |
| `LifecycleManager.java` | Injected `EodSequencer`; added `checkEodPhasing()`, `getLastEodPhase()`, `shutdown()` reset |
| `CoreConfiguration.java` | Added `eodSequencer` Spring bean; updated `lifecycleManager` bean |
| `odin-core.properties` | Added `odin.core.eod.*` defaults (15/10/5 minutes) |
| `LifecycleManagerTest.java` | Added `EodSequencer` mock |

---

## 2. Architecture Decisions

### EodSequencer as separate POJO (not in LifecycleManager)
Keeping EOD logic in a dedicated class makes it independently testable and keeps `LifecycleManager` lean. The manager acts as a thin delegate via `checkEodPhasing()`.

### Phase ordering in checkAndAdvance()
Phases are checked from highest urgency (Phase 4 at close) downward. This ensures correct behavior when the system catches up across multiple missed phases on a single bar.

### forceState() in Phase 4
Phase 4 uses `forceState(EOD)` to bypass FSM validation — consistent with kill-switch conventions. At RTH close, we need a guaranteed hard stop regardless of current pipeline state.

### Seconds precision for phase triggers
Phase thresholds use `Duration.toSeconds()` instead of `toMinutes()` to avoid up-to-59-second early activation from minute truncation.

---

## 3. ChatGPT Review (Round 1 + Round 2)

### Round 1 Findings

| ID | Classification | Finding | Resolution |
|----|---------------|---------|-----------|
| F1 | KRITISCH | Phase 4 cancels orders only for positioned pipelines — stale orders remain | **Fixed**: Phase 4 now calls `cancelAllOrders()` for ALL pipelines unconditionally |
| F2 | KRITISCH | Late-start (system restart after T-15): Phase 2/3/4 don't enforce FORCED_CLOSE | **Fixed**: `ensureForcedClose()` helper called at the start of Phase 2, 3, 4 |
| F3 | KRITISCH | Phase 2 cancels orders only for positioned pipelines — pending entry orders not cancelled | **Fixed**: Phase 2 now calls `cancelAllOrders()` for ALL pipelines first, then submits exit only if positioned |
| F4 | KRITISCH | No per-pipeline try/catch — one OMS exception aborts the entire phase loop | **Fixed**: Per-pipeline try/catch added to all 4 phase execution loops; phase still marked as executed even with failures |
| F5 | KRITISCH | cancelAllOrders() before submitExit() removes protective stops (async IB risk) | **Documented**: This is an IB TWS async interaction concern at the OMS layer, outside EodSequencer scope. Documented in JavaDoc. |
| F6 | WICHTIG | "Phase runs once" — late fills after Phase 2 not re-handled | **Accepted**: F2+F3 fixes reduce this risk significantly. Full idempotent enforcement would require per-pipeline attempt tracking (v2 scope). |
| F7 | WICHTIG | `Duration.toMinutes()` truncation — up to 59s early trigger | **Fixed**: Changed to `Duration.toSeconds()` with `threshold * 60` comparison |
| F8 | WICHTIG | `getSessionBoundaries()` not defensively handled | **Accepted**: NPE from MarketClock is a system-level failure; existing exception handling in `checkAndAdvance` propagates to caller. |
| F9 | WICHTIG | Phase 4 unresolved positions: only logged, no KillSwitch escalation | **Accepted for v1**: ERROR log + UNRESOLVED_POSITION count in EventLog payload. KillSwitch escalation is a v2 enhancement (caller can check `getLastEodPhase()` + unresolved count). |
| F10 | WICHTIG | EventLog entries missing runId | **Accepted**: EodSequencer is a session-agnostic POJO; adding runId would require dependency injection of RunContext (v2 scope). |
| F11 | HINWEIS | JavaDoc says "not a Spring Bean" but it is wired by CoreConfiguration | **Fixed**: JavaDoc updated to "Instantiated as a Spring singleton by CoreConfiguration" |
| F12 | HINWEIS | Event order test uses `atLeastOnce()` instead of `times(1)` + InOrder | **Fixed**: Test refactored to use `times(1)` + `InOrder` for exact-once and order verification |

### Round 2 Key Decisions

- **F1**: Phase 4 = "Freeze + Audit only" — no new market orders after RTH close (IB would reject or route to after-hours). `cancelAllOrders()` unconditionally, then EOD state.
- **F2**: `ensureForcedClose()` uses `transition()` first, falls back to `forceState()` if FSM rejects. Late-start is logged explicitly for audit.
- **F3**: Minimal fix confirmed: cancel all orders first, then submit exit only if positioned.
- **F4**: Per-pipeline try/catch with failure count in EventLog payload (aggregated, not per-exception spam).
- **F7**: `Duration.toSeconds()` comparison confirmed as correct fix.
- **F9**: ERROR log + EventLog `unresolvedPositions` field for v1. KillSwitch escalation deferred.

---

## 4. Gemini Review (3 Dimensions)

### Dimension 1 — Code Bugs, Error Handling (independently confirmed)
- **KRITISCH**: Pipeline loop exception isolation — confirmed and fixed (same as F4)
- **KRITISCH**: Late-start recovery — confirmed and fixed (same as F2)
- **HINWEIS**: Minute truncation precision — confirmed and fixed (same as F7)

### Dimension 2 — Konzepttreue
- **WICHTIG**: HALTED pipelines with positions in Phase 2/3 — `isActiveTradingState()` correctly excludes HALTED. The system operator must handle HALTED pipelines with open positions via kill switch. This is documented in the `isActiveTradingState()` JavaDoc.
- **WICHTIG**: Async cancel + immediate submit — same as F5; OMS-layer concern, documented.

### Dimension 3 — Production Gaps (IB TWS)
- **KRITISCH**: Async cancel + market order double-fill risk at IB — OMS-layer concern. EodSequencer correctly orchestrates the sequence; the OMS must implement cancel-confirmation before new order submission. Documented in EodSequencer JavaDoc.
- **WICHTIG**: TWS disconnect during EOD window — IB broker-level safety net (MOC/GTC orders) is an OMS/broker concern outside EodSequencer scope.
- **HINWEIS**: Market orders at T-0 may be rejected at RTH close — Phase 3 (T-5) is the correct last window for market orders.

---

## 5. New Tests Added (Post-Review)

| Test | Regression for |
|------|---------------|
| `phase1PipelineFailureIsolated` | F4 |
| `phase2CancelsAllOrdersEvenWhenNotPositioned` | F3 |
| `phase2EnforcesForcedCloseOnLateStart` | F2 |
| `phase2PipelineFailureIsolated` | F4 |
| `phase3EnforcesForcedCloseOnLateStart` | F2 |
| `phase4CancelsOrdersForAllPipelinesRegardlessOfPosition` | F1 |
| `phase4PipelineFailureIsolated` | F4 |
| `eodEventsLoggedExactlyOnceInOrder` (refactored) | F12 |

---

## 6. Test Results

```
Tests run: 169, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

- 23 EodSequencer unit tests
- 4 EodSequencer integration tests (real SimClock)
- 8 LifecycleManager tests (updated)
- 134 existing tests (unchanged, all green)

---

## 7. Acceptance Criteria Status

| Criterion | Status |
|-----------|--------|
| Phase 1 (T-15): pipelines → FORCED_CLOSE, entries blocked | Done |
| Phase 2 (T-10): cancel all orders + market exit for open positions | Done |
| Phase 3 (T-5): emergency market close for remaining positions | Done |
| Phase 4 (T-0): all orders cancelled, all pipelines → EOD | Done |
| Late-start safety: FORCED_CLOSE enforced in Phase 2/3/4 | Done |
| Per-pipeline error isolation (no cascade failure) | Done |
| Second-precision phase triggers | Done |
| EventLog events: EOD_PHASE_1/2/3, EOD_COMPLETE | Done |
| `odin.core.eod.*` configuration with cross-field validation | Done |
| `LifecycleManager.checkEodPhasing()` integration | Done |
| Unit tests (22+) + Integration tests (4+) | Done (27 EOD tests total) |
| FSM guard: FORCED_CLOSE blocks new entries | Done (existing FSM, tested) |

---

## 8. Open Items (v2 Scope)

| Item | Priority | Notes |
|------|----------|-------|
| KillSwitch escalation on unresolved positions after Phase 4 | WICHTIG | Caller can check via `getLastEodPhase()` + parse EventLog payload |
| Idempotent Phase 2/3 retry for late fills | WICHTIG | Requires per-pipeline attempt tracking |
| RunId in EOD EventLog entries | HINWEIS | Requires RunContext injection into EodSequencer |
| OMS: cancel-confirm before market order submit | KRITISCH | IB-specific async concern, OMS layer responsibility |
