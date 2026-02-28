# ODIN-030 Implementation Protocol
# Session Boundary Lifecycle Integration

**Story:** ODIN-030 — Session Boundary Lifecycle Integration
**Module:** odin-core
**Date:** 2026-02-23
**Status:** COMPLETE — 251 tests, 0 failures

---

## Summary

Implemented full session phase lifecycle integration in `odin-core`. The `LifecycleManager` now detects and reacts to all 6 `SessionPhase` transitions driven by a new `SessionPhaseMonitor` singleton. All trading day phases (PRE_MARKET → RTH_OPENING → CORE_TRADING → POWER_HOUR → RTH_CLOSE_COUNTDOWN → AFTER_HOURS) are handled with correct pipeline state transitions, audit logging, and orderly shutdown.

---

## Files Created

### New Classes

| File | Description |
|------|-------------|
| `odin-core/src/main/java/de/its/odin/core/service/SessionPhaseMonitor.java` | Spring singleton. Resolves `SessionPhase` from `MarketClock` + `SessionBoundaries` using only `odin-api` types. Tracks last known phase, polls for changes. No `EventLog` dependency — logs via SLF4J only. |

### New Test Classes

| File | Description |
|------|-------------|
| `odin-core/src/test/java/de/its/odin/core/service/SessionPhaseMonitorTest.java` | 34 unit tests covering all 6 phases, transitions, boundary precision, state management. |
| `odin-core/src/test/java/de/its/odin/core/service/LifecycleManagerSessionPhaseIntegrationTest.java` | 11 integration tests: real `SessionPhaseMonitor` + mutable clock, full day lifecycle. |

---

## Files Modified

### Production Code

| File | Changes |
|------|---------|
| `odin-core/src/main/java/de/its/odin/core/config/CoreProperties.java` | Added `SessionProperties` nested record with `@Min(0)` validated fields: `openingDurationMinutes`, `powerHourMinutes`, `closeCountdownMinutes`, `powerHourConservativeMode`. Added `@NotNull @Valid SessionProperties session` to main record. |
| `odin-core/src/main/java/de/its/odin/core/service/LifecycleManager.java` | Added `sessionPhaseMonitor` field + constructor param. Added `checkSessionPhase()`, `getCurrentSessionPhase()`, `logSessionPhaseChanged()`, 6 private phase handlers, `reconcilePipelineStateOnStart()`. Updated `startTrading()` with AFTER_HOURS guard + late-start reconciliation. Updated `shutdown()` to reset monitor. |
| `odin-core/src/main/java/de/its/odin/core/config/CoreConfiguration.java` | Added `sessionPhaseMonitor()` @Bean. Updated `lifecycleManager()` @Bean to accept `SessionPhaseMonitor`. |
| `odin-core/src/main/resources/odin-core.properties` | Added session phase monitoring config block (`odin.core.session.*`). |

### Test Code

| File | Changes |
|------|---------|
| `odin-core/src/test/java/de/its/odin/core/service/LifecycleManagerTest.java` | Added `sessionPhaseMonitor` mock. Added 10 new tests for `checkSessionPhase`, late-start reconciliation, AFTER_HOURS guard. Updated constructor call and AFTER_HOURS test stubs. |

---

## Key Design Decisions

### 1. SessionPhaseMonitor — odin-api Only

`SessionPhaseMonitor` uses only `MarketClock` + `SessionBoundaries` from `odin-api`, not `SessionPhaseResolver` from `odin-data`. This avoids coupling odin-core lifecycle config to odin-data's `DataProperties` structure, and keeps the monitor lean. Both implementations use identical phase resolution logic.

### 2. RTH_OPENING = Calibration Window (No WARMUP→OBSERVING)

Critical semantic: `RTH_OPENING` (09:30 to 09:30+`openingDurationMinutes`) is the calibration window. No entries are allowed. WARMUP pipelines remain in WARMUP. The WARMUP→OBSERVING transition fires only when `CORE_TRADING` begins (calibration ends). This correctly models concept 03 section 1.

### 3. AFTER_HOURS = Full stopTrading() Delegation

`handleAfterHoursPhase()` calls `stopTrading(new ArrayList<>(activePipelines))` rather than manually forcing FSM states. This ensures: data pipeline shutdown, market data unsubscription, order cancellation, FSM→EOD transition, and EOD performance recording — complete orderly shutdown.

### 4. SESSION_PHASE_CHANGED Audit Logging in LifecycleManager

`SessionPhaseMonitor` no longer has an `EventLog` dependency. All `SESSION_PHASE_CHANGED` events are logged by `LifecycleManager.logSessionPhaseChanged()` which has access to `currentRunContext.runId()`. This ensures proper run correlation in the audit trail. `SessionPhaseMonitor` uses SLF4J only.

### 5. Late-Start Reconciliation

`startTrading()` calls `reconcilePipelineStateOnStart()` after subscription. If `resolveCurrentPhase()` returns CORE_TRADING, POWER_HOUR, or RTH_CLOSE_COUNTDOWN, any WARMUP pipeline is immediately transitioned to OBSERVING. Prevents "stuck in WARMUP forever" bug when pipelines join a session that has already advanced past the opening calibration.

### 6. AFTER_HOURS Guard in startTrading()

`startTrading()` uses `sessionPhaseMonitor.resolveCurrentPhase()` (not cached `getLastKnownPhase()`) to guard against starting during AFTER_HOURS. Using the live-resolved phase ensures correctness even if `checkSessionPhase()` has never been called (null lastKnownPhase) or the clock has advanced past the last polled phase.

---

## ChatGPT Review Summary (2 Rounds)

### Round 1 Findings → Actions

| Finding | Priority | Action |
|---------|----------|--------|
| RTH_OPENING semantics wrong (WARMUP→OBSERVING too early) | KRITISCH | FIXED: moved to `handleCoreTradingPhase()` |
| AFTER_HOURS doesn't properly stop pipelines | KRITISCH | FIXED: `stopTrading()` delegation |
| odin-core imports odin-data | KRITISCH | DISMISSED: pre-existing by design |
| SESSION_PHASE_CHANGED lacks runId | WICHTIG | FIXED: logging moved to LifecycleManager |
| RTH_CLOSE_COUNTDOWN delegation implicit | WICHTIG | ACCEPTED: by design |
| PRE_MARKET not enforcing initialization | WICHTIG | ACCEPTED: external startup responsibility |
| Missing validation for overlapping windows | WICHTIG | DEFERRED: @Min(0) only for v1 |

### Round 2 Findings → Actions

| Finding | Priority | Action |
|---------|----------|--------|
| Late-start "CORE_TRADING already active" → stuck in WARMUP | KRITISCH | FIXED: `reconcilePipelineStateOnStart()` |
| `getLastKnownPhase()` can be null before first poll | WICHTIG | FIXED: fallback to `resolveCurrentPhase()` in reconciliation |
| AFTER_HOURS guard when startTrading called too late | WICHTIG | FIXED: `resolveCurrentPhase()` guard in `startTrading()` |
| Stale JavaDoc (pollPhaseChange, class-level bullets) | WICHTIG | FIXED: updated all affected JavaDoc |
| "09:45" hard-coded time reference in comments | HINWEIS | FIXED: replaced with config-driven description |

---

## Gemini Review Summary (1 Round)

Overall assessment: **"Highly polished, robust, and mature engineering."**

| Dimension | Assessment |
|-----------|------------|
| Code Quality | Excellent. No var, explicit types, MarketClock used consistently, Records, JavaDoc, verb-first naming. |
| Konzepttreue | Accurate. Phase resolution correctly models 6-phase lifecycle. RTH_OPENING/CORE_TRADING split correct. AFTER_HOURS shutdown correct. EventLog split (Monitor=SLF4J, LifecycleManager=EventLog+runId) architecturally sound. |
| Produktionstauglichkeit | Strong. Non-fatal error handling, late-start reconciliation, thread safety documentation, volatile + List.copyOf() defensive patterns. |
| Minor Observation | `activePipelines` could be captured as local var before iteration (defensive snapshot). Accepted: documented as single-threaded. |

---

## Configuration Added

```properties
# odin-core.properties
odin.core.session.opening-duration-minutes=15
odin.core.session.power-hour-minutes=30
odin.core.session.close-countdown-minutes=15
odin.core.session.power-hour-conservative-mode=false
```

---

## Test Results

| Test Class | Tests | Status |
|------------|-------|--------|
| `LifecycleManagerTest` | 20 | GREEN |
| `LifecycleManagerSessionPhaseIntegrationTest` | 11 | GREEN |
| `SessionPhaseMonitorTest` (all nested) | 34 | GREEN |
| All other odin-core tests | 186 | GREEN |
| **TOTAL** | **251** | **BUILD SUCCESS** |
