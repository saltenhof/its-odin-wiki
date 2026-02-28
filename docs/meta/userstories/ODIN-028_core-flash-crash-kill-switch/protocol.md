# ODIN-028 — Protocol: Flash Crash Detection & Kill-Switch Enhancement

**Story:** Flash Crash Detection und Kill-Switch Enhancement
**Module:** odin-core
**Status:** IMPLEMENTATION COMPLETE — ready for QS-Agent review
**Date:** 2026-02-23

---

## 1. Implementation Summary

### New Files
- `odin-api/src/main/java/de/its/odin/api/model/KillSwitchTrigger.java` — Enum classifying kill-switch activation causes: FLASH_CRASH, DAILY_DRAWDOWN, HEARTBEAT_TIMEOUT, DQ_ESCALATION, BROKER_EMERGENCY, MANUAL
- `odin-core/src/test/java/de/its/odin/core/service/KillSwitchServiceIntegrationTest.java` — Integration tests for KillSwitchService + GlobalRiskManager

### Modified Files
| File | Change |
|------|--------|
| `odin-core/.../KillSwitchService.java` | Complete rewrite: 3 automatic triggers, idempotency, EventLog audit, NaN guards |
| `odin-core/.../GlobalRiskManager.java` | Added `computeCombinedDailyDrawdownPct()`, fixed `onExitFill` unrealizedPnl reset |
| `odin-core/.../CoreProperties.java` | Added `KillSwitchProperties` record with 5 config fields |
| `odin-core/.../CoreConfiguration.java` | Updated `killSwitchService()` bean factory (3-arg constructor) |
| `odin-core/src/main/resources/odin-core.properties` | Added 5 new kill-switch config defaults |
| `odin-core/.../KillSwitchServiceTest.java` | Complete rewrite: 35 tests covering all triggers, edge cases, NaN/Infinity, EventLog idempotency |
| `odin-core/.../GlobalRiskManagerTest.java` | Added 6 new tests for computeCombinedDailyDrawdownPct + onExitFill unrealized reset |
| `odin-backtest/.../BacktestRunner.java` | Fixed pre-existing bugs (CoreProperties constructor, KillSwitchService constructor, LifecycleManager constructor) |
| `odin-backtest/.../BacktestRunnerTest.java` | Updated CoreProperties constructor to 7-arg |
| `odin-backtest/.../BacktestRunnerGovernanceIntegrationTest.java` | Updated CoreProperties constructor to 7-arg |
| `odin-app/.../ControlControllerTest.java` | Updated KillSwitchService constructor |
| `odin-app/.../ExchangeControllerTest.java` | Updated CoreProperties + PipelineProperties constructors |
| `odin-app/.../TradingRunControllerTest.java` | Updated CoreProperties + PipelineProperties constructors |

---

## 2. Design Decisions

### 2.1 Idempotency Implementation
**Decision:** `volatile boolean killed` for lock-free hot-path reads; `synchronized` for all mutation.

**Rationale:** `isKilled()` is called on every bar (~1s decision loop). Volatile ensures cache-coherent reads without synchronization overhead. Mutation (`activateKillSwitch`) is rare and lock-protected. The combination is race-condition-free: `synchronized void activateKillSwitch()` checks `this.killed` inside the lock before setting, so only the first thread through the lock activates the kill switch.

### 2.2 Return Value Semantics
**Decision:** `checkFlashCrash()`, `checkDailyDrawdown()`, `checkHeartbeat()` all return `true` only when the kill switch transitions from inactive to active in this call.

**Rationale:** Uniform semantics: `true` = "this call triggered the kill switch". When already killed, early-exit `if (killed) return false` prevents lock contention, log spam, and unnecessary EventLog writes.

### 2.3 Flash Crash Direction-Agnostic
**Decision:** `checkFlashCrash` checks absolute price move, not directional (drop only).

**Rationale:** The story says ">5% Preisbewegung" (price movement), not "Kursrückgang" (drop). The odin-data CrashDetectionGate is responsible for event classification; KillSwitchService only decides. Direction-awareness belongs at the data layer per the "odin-data eskaliert, odin-core entscheidet" ownership principle.

### 2.4 Threshold Comparison: >= vs >
**Decision:** `>=` (greater-or-equal) for all threshold comparisons.

**Rationale:** The story uses ">5%" and ">10%", but floating-point precision makes strict `>` unreliable at boundary values. `>=` is the conservative choice (earlier trigger = safer for risk management). Documented in Javadoc: "A price drop of exactly the threshold triggers the kill switch."

### 2.5 EventLog in Synchronized Block
**Decision:** `eventLog.append()` is called inside the `synchronized activateKillSwitch()` block.

**Rationale:** In the ODIN Live setup, `PostgresEventLog` delegates to `AuditEventDrainer`, which uses a bounded async queue. The `append()` call is non-blocking in practice. The synchronized block is held for milliseconds at most (rare event). Moving EventLog outside the lock would introduce a TOCTOU gap where the audit event might be logged without holding the activation invariant. The current design is simpler and correct for ODIN's concurrency model.

### 2.6 Heartbeat: Counter-Based vs Time-Based
**Decision:** Counter-based (count consecutive scheduler invocations without recordHeartbeat()).

**Rationale:** The story specifies "3 aufeinanderfolgende verpasste Heartbeats" — consecutive misses, not elapsed time. The scheduler controls the 30s interval; the counter counts misses in that cadence. A time-based implementation would require storing `lastHeartbeatTime` and checking elapsed duration — more complex, but more GC-resilient. Documented as known limitation below.

---

## 3. Bug Fix: GlobalRiskManager.onExitFill — Stale Unrealized PnL

**Bug:** When a position was closed via `onExitFill()`, the `unrealizedPnl` field was not reset to 0. This caused stale unrealized losses to persist in `computeCombinedDailyDrawdownPct()`, inflating the drawdown calculation and potentially triggering the kill switch prematurely.

**Fix:** Added `state.unrealizedPnl = 0.0` in `onExitFill()` after setting `positioned = false`.

**Test:** `onExitFillShouldResetUnrealizedPnlToZero` in `GlobalRiskManagerTest`.

---

## 4. Pre-Existing Bugs Fixed (BacktestRunner)

Two pre-existing compilation bugs were found and fixed in `BacktestRunner.java`:

1. `CoreProperties` constructor called with 5 args (missing `eod` and `killSwitch`) → fixed to 7-arg
2. `new KillSwitchService()` with no-arg constructor (doesn't exist) → fixed with 3-arg constructor
3. `LifecycleManager` constructor missing `eodSequencer` arg → fixed by instantiating `EodSequencer` and passing it

Additionally, `BacktestRunnerTest`, `BacktestRunnerGovernanceIntegrationTest`, `ExchangeControllerTest`, and `TradingRunControllerTest` all had stale `CoreProperties` constructor calls — all fixed.

---

## 5. ChatGPT Review (2 Rounds)

### Round 1 Findings and Disposition

| Finding | Severity | Action |
|---------|----------|--------|
| Flash crash direction-agnostic (no drop-only) | KRITISCH | Rejected — story says "Preisbewegung", not "Rückgang"; ownership principle |
| No early-exit `if (killed) return false` in checkFlashCrash/checkDailyDrawdown | KRITISCH | **Fixed** |
| Stale unrealizedPnl after Exit | KRITISCH | **Fixed** in GlobalRiskManager.onExitFill |
| EventLog in synchronized block (SLO risk) | KRITISCH | Rejected — EventLog is non-blocking (AuditEventDrainer queue); documented as design decision |
| Return value semantics inconsistent | WICHTIG | **Fixed** (same as early-exit fix) |
| Package refactor to `de.its.odin.core.kill-switch` | WICHTIG | Rejected — "Namespace" in story = config namespace, not Java package; existing convention is `de.its.odin.core.service` |
| Javadoc Instant signature mismatch | WICHTIG | **Fixed** |
| Config units inconsistency (percent vs fraction) | WICHTIG | Documented as tech-debt; pre-existing `dailyHardStopPercent` is out of ODIN-028 scope |
| Missing `@Max(1)` on flashCrashThresholdPct | HINWEIS | **Fixed** |
| Redundant `SECONDS_PER_MISS` constant | HINWEIS | **Fixed** — removed |

### Round 2 Findings and Disposition

| Finding | Action |
|---------|--------|
| EventLog `times(1)` verify test | **Added** — `eventLogShouldBeCalledExactlyOnceOnFirstActivation` + `eventLogShouldNotBeCalledOnSubsequentActivationAttempts` |
| NaN/Infinity guards in checkFlashCrash/checkDailyDrawdown | **Added** — defensive guards with WARN log |
| Tests for NaN/Infinity inputs | **Added** — 5 tests in KillSwitchServiceTest |

---

## 6. Gemini Review (3 Dimensions)

### Dimension 1: Code Bugs, Thread-Safety, Latency SLO

| Finding | Assessment |
|---------|-----------|
| GlobalRiskManager lock-contention (all methods synchronized on `this`) | Valid concern for HFT; ODIN has 2-3 instruments and 1s bars — not HFT. Pre-existing; out of ODIN-028 scope. |
| Idempotency guard (volatile + synchronized) is race-condition-free | Confirmed correct |
| EventLog in synchronized block | Same as ChatGPT finding — accepted as design decision |
| Heartbeat AtomicInteger race | Minor and acceptable; consequence is at most one phantom miss |
| NaN/division by zero handling | Already implemented (referencePrice <= 0, NaN guards) |

### Dimension 2: Concept Compliance

| Check | Result |
|-------|--------|
| Flash crash `>=` vs `>` threshold | `>=` implemented — conservative, documented in Javadoc |
| Realized + unrealized losses correctly summed (gains don't offset) | Confirmed correct |
| Heartbeat 3x30s logic | Confirmed correct |
| Kill-switch action sequence (Cancel All → Market Close → DAY_STOPPED → EMERGENCY) | Pull model confirmed — Pipeline polls `isKilled()`, executes wind-down. Architecture is correct. |
| Latency SLO measurement via EventLog timestamp | Confirmed correct (`triggerReceivedAt = marketClock.now()` at activation) |

### Dimension 3: Production Gaps

| Gap | Status |
|-----|--------|
| Kill-Switch during EOD sequence → double-close risk | Known gap — Pipeline must check OMS state before firing Market-Close. Out of ODIN-028 scope; documented as open point. |
| TWS disconnect during Cancel-All → orphaned orders | Mitigated by IB TWS "Cancel Active Orders on Disconnect" server setting. ODIN must activate this. Out of scope. |
| Heartbeat false positive from GC pause → counter fires instantly | **Known Limitation** (see below) |
| Concurrent Kill-Switch + Degradation | No gap — synchronized activation handles this correctly |
| Drawdown reset state bleed | No gap — JVM restart daily; reset() is simulation-only |

---

## 7. Known Limitations and Open Points

### KL-001: Heartbeat GC False Positive
**Description:** If the Heartbeat scheduler thread is stalled by a Stop-the-World GC event for >90s and then fires its 3 pending ticks in rapid succession, the kill switch triggers without an actual market data loss.

**Current behavior:** Counter-based (counts scheduler invocations). 3 rapid calls = 3 misses = trigger.

**Proposed fix (not in scope):** Change to time-based: store `lastHeartbeatAt = marketClock.now()` in `recordHeartbeat()`, and in `checkHeartbeat()` compare `Duration.between(lastHeartbeatAt, now)` against `heartbeatMissCount * heartbeatIntervalSeconds`. If elapsed time < threshold, don't trigger.

**Risk level:** Low. ODIN runs on a dedicated Windows server; G1GC with modest heap rarely exceeds 1-2s STW pauses. 90s GC pause would indicate a serious JVM misconfiguration, which is itself an operational emergency.

### KL-002: EOD + Kill-Switch Double-Close Race
**Description:** If Kill-Switch fires while the Pipeline is already in EXITING state with market-close orders pending, blindly firing another set of market-close orders could briefly invert the position.

**Resolution:** The OMS must check position state before submitting emergency exits. Out of ODIN-028 scope — tracked for ODIN-030 (OMS wind-down sequencing).

### KL-003: Config Units Inconsistency
**Description:** `GlobalRiskProperties.dailyHardStopPercent` uses 100-scale (e.g., `5.0` = 5%), while `KillSwitchProperties.drawdownThresholdPct` uses fraction scale (e.g., `0.10` = 10%). This is a misconfiguration risk.

**Resolution:** Pre-existing; would require a breaking config change. Separate tech-debt ticket.

---

## 8. Test Coverage Summary

| Test Class | Tests | Scope |
|------------|-------|-------|
| KillSwitchServiceTest | 35 | Unit: all triggers, idempotency, NaN guards, EventLog verify |
| KillSwitchServiceIntegrationTest | 7 | Integration: KillSwitchService + GlobalRiskManager |
| GlobalRiskManagerTest | 25 | Unit: existing + drawdownPct + unrealizedPnl reset |

**Total new/modified tests:** 67 (across all test classes)
**Full test suite:** 219 tests (odin-app) + 205 tests (odin-core) — all GREEN

---

## 9. Config Defaults Added

```properties
odin.core.kill-switch.flash-crash-threshold-pct=0.05
odin.core.kill-switch.flash-crash-volume-multiplier=3.0
odin.core.kill-switch.drawdown-threshold-pct=0.10
odin.core.kill-switch.heartbeat-miss-count=3
odin.core.kill-switch.heartbeat-interval-seconds=30
```

---

## 10. Acceptance Criteria Checklist

- [x] Flash Crash: Preis ändert sich >5% UND Volume > 3x → Kill-Switch (`checkFlashCrash`)
- [x] Tages-Drawdown: Summe(realized + unrealized) > 10% → Kill-Switch (`checkDailyDrawdown` + `computeCombinedDailyDrawdownPct`)
- [x] Heartbeat: 3 aufeinanderfolgende verpasste Heartbeats (90s total) → Kill-Switch (`checkHeartbeat`)
- [x] Kill-Switch-Latenz: < 2 Sekunden messbar via EventLog-Timestamps (`triggerReceivedAt` + EventLog `KILL_SWITCH_TRIGGERED`)
- [x] Kill-Switch-Aktion: Pipelines prüfen `isKilled()` und führen Cancel All → Market-Close → DAY_STOPPED → EMERGENCY aus
- [x] Kill-Switch ist idempotent (mehrfacher Aufruf: erste Ursache bleibt, weitere = No-Op)
- [x] Kein Re-Entry nach Kill-Switch für den Rest des Tages
- [x] `KillSwitchTrigger` Enum in odin-api
- [x] Config-Namespace `odin.core.kill-switch.*`
- [x] Alle Tests grün (mvn test = BUILD SUCCESS)
- [x] ChatGPT-Review: 2 Runden durchgeführt
- [x] Gemini-Review: 3 Dimensionen durchgeführt
