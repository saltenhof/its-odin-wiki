# ODIN-007: Protocol — 1-Minute Monitor Event Detection

**Story:** ODIN-007
**Status:** DONE
**Date:** 2026-02-21
**Module:** odin-data

---

## Working State

| Milestone | Status | Notes |
|-----------|--------|-------|
| odin-api: MonitorEventType, MonitorEvent, MonitorEventListener | DONE | 9 types, immutable record event, listener port |
| DataProperties.MonitorProperties (11 fields) | DONE | Nested record, validated, odin.data.monitor.* namespace |
| MonitorEventDetector: 9 detectors + cooldown | DONE | POJO, not a Spring bean |
| DataPipelineService integration | DONE | evaluate() called post-warmup on each 1m bar |
| Unit tests (268 total, MonitorEventDetector focused) | DONE | All pass |
| Integration tests (11 total, 3 new for ODIN-007) | DONE | All pass |
| ChatGPT Review | DONE | Multiple rounds, critical findings fixed |
| Gemini Review | DONE | Confirmed ChatGPT findings, no new criticals |
| Post-review fixes | DONE | Compiled and tested |

---

## Design Decisions

### 1. Event type mapping vs. story scope
The story requested "Volume Spike, Price Break Level, Momentum Divergence, Spread Expansion, Stale Quote Warning, Unusual Activity, Range Expansion, VWAP Cross, Exhaustion Signal" but the existing `MonitorEventType` enum (from ODIN-005) defines: CRASH_DOWN, SPIKE_UP, EXHAUSTION_CLIMAX, STRUCTURE_BREAK_1M, DQ_STALE, DQ_OUTLIER, DQ_GAP, VWAP_CROSS_1M, EMA_CROSS_1M. The implementation maps to the established enum, which is the source of truth. The story names were conceptual labels, not final type names.

### 2. Exhaustion Signal: simple proxy
The 3-pillar Exhaustion Signal (RSI divergence + volume climax + wick pattern) is complex and slated for ODIN-015. ODIN-007 implements a simple proxy: large upper wick ratio > `exhaustionWickRatio` AND volume > `exhaustionVolumeMultiplier * SMA`. This is explicitly documented in both the JavaDoc and the detector implementation.

### 3. EMA_CROSS_1M: SMA proxy
The detector is named EMA_CROSS but uses a simple SMA of recent closes (not a true exponential MA). This is a conscious lightweight proxy — no ta4j dependency in odin-data, and the full EMA is available from odin-brain's KPI engine. Documented in JavaDoc as "SMA proxy (full EMA in ODIN-015)".

### 4. Cooldown state: central Map, not per-detector
A single `Map<MonitorEventType, Instant> lastEmissionByType` (EnumMap) is used rather than embedding cooldown state in each detector method. This keeps the evaluate() loop clean and makes cooldown logic trivially inspectable in tests.

### 5. Rolling state update ordering (critical — post-review fix)
Originally, `updateVolumeHistory()` and `updateRecentCloses()` were called BEFORE the detectors ran. This caused the current bar's volume/close to be included in the SMA baseline used by the current bar's detectors — self-inclusion bias. After ChatGPT/Gemini review, `evaluate()` was restructured to run all 9 detectors first, then advance rolling state. This ensures each bar is compared against a strictly prior-bar baseline.

### 6. MonitorEventListener registration
`DataPipelineService.registerMonitorEventListener(MonitorEventListener)` is a post-construction registration method, not constructor injection. This matches the existing pattern for `DataPipelineListener` and allows the caller (odin-core's PipelineFactory) to wire the listener after pipeline construction.

### 7. DQ_STALE threshold: 2x standard interval + staleQuoteWarningS
The stale detection uses two conditions: the bar gap must exceed both 120 seconds (2x the normal 1m interval, to avoid false positives on minor timing jitter) AND `staleQuoteWarningS` from config. In bar-replay/simulation mode, bar time gaps serve as a staleness proxy.

### 8. Listener exception safety
`emitIfCooldownExpired()` wraps the `listener.onMonitorEvent()` call in try/catch (RuntimeException). A faulty listener must not crash the pipeline processing thread. The exception is logged at ERROR level and suppressed.

---

## ChatGPT Sparring Summary

Two ChatGPT review rounds were conducted via the ChatGPT Session Pool Service.

### Round 1: Initial code review

ChatGPT identified the following findings:

| # | Severity | Finding | Decision |
|---|----------|---------|----------|
| 1 | CRITICAL | Volume SMA self-inclusion: current bar volume added before detection inflates the threshold baseline | FIXED — reordered evaluate() |
| 2 | CRITICAL | EMA_CROSS SMA self-inclusion: same issue for recentCloses | FIXED — reordered evaluate() |
| 3 | IMPORTANT | DQ_STALE hardcoded `SECONDS_PER_MINUTE * 2 = 120s` — not derived from config | ACCEPTED as is — the 2x cap is an intentional design guard, not a config concern. `staleQuoteWarningS` is still applied as second condition |
| 4 | IMPORTANT | Listener exception safety — uncaught RuntimeException from listener crashes pipeline | FIXED — added try/catch in emitIfCooldownExpired() |
| 5 | CLEANUP | Dead code: `rangeHistory`, `ATR_BASELINE_WINDOW`, `MIN_BARS_FOR_DIVERGENCE`, `RSI_DIVERGENCE_DROP_THRESHOLD`, `baselineSpread`, `spreadSampleCount`, `MIN_SPREAD_SAMPLES`, `SECONDS_PER_MINUTE`, `updateRangeHistory()`, `updateSpreadBaseline()`, `computeAtrBaseline()` | FIXED — all removed |
| 6 | MINOR | Integration test JavaDoc referenced "VOLUME_SPIKE" (wrong type name) | FIXED |
| 7 | MINOR | Class JavaDoc: "EMA_CROSS_1M — volume-weighted SMA proxy" is inaccurate | FIXED — corrected to "simple SMA proxy" |

### Key Insight from ChatGPT
The self-inclusion fix is not just correctness — it changes the semantics. With self-inclusion, the SMA threshold was systematically higher on every bar (the bar was compared against a baseline that included itself). The fix ensures the detector sees: "is this bar's volume anomalously high vs. all prior bars?" rather than "is this bar's volume anomalously high vs. a baseline that includes this bar?".

---

## Gemini Review Summary

Gemini confirmed all ChatGPT findings independently and added:

- Architecture assessment: The POJO design (no Spring dependency) and strict odin-api port usage are correct.
- Performance: 9 detectors per 1m bar is negligible overhead. No performance concern.
- No new critical findings beyond what ChatGPT identified.
- Suggestion: Consider documenting that `EXHAUSTION_CLIMAX` and `EMA_CROSS_1M` are explicit proxies in the story. Already done via JavaDoc.

---

## Test Debugging Notes

### Unit test failures (fixed before final submit)

**Failure: `cooldown_allowsEventAfterCooldownExpires`**
Root cause: After `populateVolumeHistory(500, 15)` + two bars, crash2 had volume=2000 which exactly equaled the computed SMA threshold (2000.0). Strict `>` comparison: `2000 > 2000.0 = false`.
Fix: Raised crash2 volume to 3000 to clear the floating-point boundary.

**Failure: `cooldown_differentTypesAreIndependent`**
Root cause: refBar used `snapshotWithVwap(100.0, 0.0)` (vwap=0). The VWAP guard `if (vwap <= 0) return` caused `detectVwapCross` to exit early, leaving `previousPriceAboveVwap = null`. Subsequent crashBar was the first valid VWAP bar: null → no cross → VWAP_CROSS never fired.
Fix: Changed refBar to `snapshotWithVwap(100.0, 99.0)`. With close=100 > vwap=99, `previousPriceAboveVwap` transitions null→false→true → VWAP_CROSS fires on refBar. Used `SHORT_COOLDOWN_CONFIG` (1s) to ensure the 60s gap to crashBar clears the cooldown.

### Integration test failures (fixed before final submit)

**Root cause 1: MonotonicityGate rejection**
Warmup bars were sent at RTH_OPEN+1min through RTH_OPEN+20min. Test bars were sent at times before the last warmup bar time → MonotonicityGate rejected them as out-of-order.
Fix: Introduced `FIRST_TEST_BAR_TIME = RTH_OPEN+21min` (strictly after all warmup bars).

**Root cause 2: OutlierFilterGate rejecting crash bars**
Daily bar volume was 5,000,000 → `calibratedAvgVolume = 5,000,000` → CrashDetectionGate threshold = 15,000,000. Test crash bars (volume=5000-10000) did not exceed the threshold → no CRASH_SIGNAL flag. OutlierFilterGate saw 6% price deviation without CRASH_SIGNAL → REJECT_EVENT.
Fix: Changed daily bar volume to 3,000 → `calibratedAvgVolume = 3,000` → threshold = 9,000. Crash bars with volume=10,000 correctly receive CRASH_SIGNAL and pass through OutlierFilterGate.

---

## Open Points / Follow-up

| # | Item | Story |
|---|------|-------|
| 1 | Full 3-pillar Exhaustion Signal (RSI divergence + climax volume + wick) | ODIN-015 |
| 2 | Full EMA cross using ta4j KPI engine (currently SMA proxy) | ODIN-015 |
| 3 | Spread Expansion detector (requires SpreadStats baseline; current proxy not implemented) | ODIN-015 |
| 4 | odin-core reaction to MonitorEvent (LLM trigger, Kill-Switch escalation routing) | ODIN-010+ |
