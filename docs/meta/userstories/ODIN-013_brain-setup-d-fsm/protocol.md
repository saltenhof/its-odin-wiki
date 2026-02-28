# ODIN-013: Protocol — ReEntryCorrectionFsm Implementation

**Story:** ODIN-013 brain-setup-d-fsm
**Status:** QS R1 FAIL — Fixes required (see qa-report-r1.md)
**Date:** 2026-02-22
**Agent:** Claude Sonnet 4.6

---

## 1. Implementation Summary

Implemented `ReEntryCorrectionFsm` (Setup D: Re-Entry after Correction) as a `PatternStateMachine`
in the odin-brain module.

### Files Created

- `odin-brain/src/main/java/de/its/odin/brain/rules/pattern/ReEntryCorrectionFsm.java`
- `odin-brain/src/test/java/de/its/odin/brain/rules/pattern/ReEntryCorrectionFsmTest.java`
- `odin-brain/src/test/java/de/its/odin/brain/rules/pattern/ReEntryCorrectionFsmIntegrationTest.java`

### Files Modified

- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java`
  — Added `reEntryAdxTrendThreshold` and `reEntryPullbackAtrTimeoutFactor` to `PatternProperties`
- `odin-brain/src/main/resources/odin-brain.properties`
  — Added default values: `re-entry-adx-trend-threshold=25.0`, `re-entry-pullback-atr-timeout-factor=3.0`
- `odin-brain/src/main/java/de/its/odin/brain/rules/RulesEngine.java`
  — Registered `ReEntryCorrectionFsm` in `initializePatternFsms()`
- `odin-brain/src/test/java/de/its/odin/brain/TestBrainProperties.java`
  — Updated `PatternProperties` constructor to include 2 new fields
- `odin-brain/src/test/java/de/its/odin/brain/kpi/KpiEngineTest.java`
  — Fixed broken `PatternProperties` constructor call (old 10-arg → new 12-arg)

---

## 2. Design Decisions

### Swing High Anchoring

**Decision:** `swingHighBeforePullback` is set to `currentBar.high()` at the TREND_ESTABLISHED
transition (not `snapshot.intradayHigh()`). It is continuously updated upward in TREND_ESTABLISHED
as new bars make higher highs.

**Rationale:** Using the FSM-tracked swing high (from its own higher-low series) rather than the
session intraday high ensures the Fibonacci levels are calibrated to the actual trend structure the
FSM observed. The intraday high can reflect pre-market or unrelated moves.

### Swing Low Anchoring

**Decision:** `swingLowBeforeRun = recentLows.getFirst()` (oldest tracked low in the rolling window)
at the TREND_ESTABLISHED transition.

**Rationale:** The oldest tracked low in the higher-low series represents the origin of the current
trend. Using `currentBar.low()` at transition time was tried but produces a too-small up-move range
that distorts Fibonacci levels. The rolling window (20 bars) is more than sufficient for intraday
trends that typically establish within 4-10 bars.

### ATR-Based Timeout (Close Price)

**Decision:** The ATR timeout check uses `currentPrice` (close) rather than `currentBar.low()`.

**Rationale:** In 5-min intraday bars, wicks represent intrabar noise. A spike low that recovers
to close near the highs should not invalidate an otherwise valid trend structure. The Fibonacci
breakdown check (which uses `pullbackLow`) acts as the complementary structural check for absolute
level breaches. The two checks are complementary, not redundant:
- ATR/close: catches gradual drift below the trend structure
- Fibonacci/pullbackLow: catches absolute structural breakdowns

### pullbackLow on PULLBACK_ACTIVE Entry

**Decision:** `pullbackLow = currentBar.low()` at the PULLBACK_ACTIVE transition (not `currentPrice`).

**Rationale:** The entry bar into the pullback zone may have a low that is lower than the close.
Using `currentBar.low()` correctly captures the full adverse excursion from the first pullback bar,
ensuring the stop level is set to the actual lowest point seen.

### Confidence Formula

**Decision:** Confidence is inversely proportional to pullback depth throughout the valid Fibonacci
zone (38.2%–61.8%). Shallower pullbacks (toward 38.2%) yield higher confidence than deeper ones
(toward 61.8%). The 50% level is the calibration reference point; below 50%, confidence scales
linearly up toward `CONFIDENCE_MAX` (0.85); above 50%, it interpolates down to
`CONFIDENCE_AT_61_8_PERCENT` (0.58). Clipped to [0.55, 0.85].

**Rationale:** A shallower pullback indicates a stronger trend momentum — the market doesn't need
to give back much ground before resuming. This is consistent with standard Fibonacci trading theory.

---

## 3. Bugs Fixed During Implementation

1. **currentPrice variable scope in processIdle**: Initial code referenced `currentPrice` which was
   not in scope. Fixed to `snapshot.currentPrice()`.

2. **Immediate ATR timeout on TREND_ESTABLISHED**: Initial version used `snapshot.intradayHigh()`
   as the swing high anchor, which caused the distance from current price to swing high to immediately
   exceed 3 ATR (intraday high was set to 120 while current price was ~104). Fixed by tracking the
   FSM's own swing high from `currentBar.high()`.

3. **pullbackLow set to currentPrice instead of currentBar.low()**: When entering PULLBACK_ACTIVE,
   the entry bar's low was not captured. Fixed so `pullbackLow = currentBar.low()` at transition.
   (Found during ChatGPT sparring review.)

---

## 4. Test Results

### Unit Tests (`ReEntryCorrectionFsmTest.java`)

14 tests, all passing. Coverage:
- Full FSM lifecycle (IDLE → TREND_ESTABLISHED → PULLBACK_ACTIVE → signal)
- Deep pullback timeout (> 3 ATR → IDLE)
- ADX below threshold (no TREND_ESTABLISHED)
- Insufficient higher-lows (< 3 consecutive, no TREND_ESTABLISHED)
- Confidence ordering (shallower depth → higher confidence)
- Shallow Fibonacci pullback (near 38.2% end of zone)
- EMA50 proximity support trigger (inside 0.25*ATR of EMA50)
- VWAP proximity support trigger (inside 0.25*ATR of VWAP)
- Swing-high resumption path (close > swingHigh, EMA9 not triggered)
- NaN ATR guard (no state change, no signal)
- FSM identity (patternName, initialState, emptyBars, reset)

### Integration Tests (`ReEntryCorrectionFsmIntegrationTest.java`)

3 tests, all passing. Coverage:
- Full tick stream: trend → pullback → resumption → ENTRY via RulesEngine
- Deep pullback (> 3 ATR after PULLBACK_ACTIVE) → NO_ACTION
- LLM absent (null LlmAnalysis) → LLM mandatory gate blocks entry → NO_ACTION

### Full odin-brain Test Suite

476 tests total, 0 failures.

---

## 5. ChatGPT Sparring (DoD 2.5)

**Session:** ChatGPT pool slot 0, owner `odin-013-chatgpt-sparring`

**Files reviewed:** ReEntryCorrectionFsm.java, ReEntryCorrectionFsmTest.java,
ReEntryCorrectionFsmIntegrationTest.java

**Key findings and resolutions:**

| Finding | Resolution |
|---------|-----------|
| EMA50 proximity support trigger not tested | Added `ema50ProximitySupport_triggersTransitionToPullbackActive` test |
| VWAP proximity support trigger not tested | Added `vwapProximitySupport_triggersTransitionToPullbackActive` test |
| "Close > swingHigh" resumption path not tested | Added `resumptionAboveSwingHigh_emitsSignalWithoutEma9Condition` test |
| pullbackLow set to currentPrice at PULLBACK_ACTIVE entry (Bug B) | Fixed: `pullbackLow = currentBar.low()` |
| Class JavaDoc said "50% = highest confidence" but code gives higher for shallower | Updated class + method JavaDoc to correctly describe monotonic inverse relationship |
| Integration test used intraday high/low as FSM swing anchors (incorrect assumption) | Rewrote integration test with correct price setup tracking FSM's own swing anchors |
| Over-coupling to default config values | Acceptable: TestBrainProperties constants are explicit, test is self-documenting |
| Intrabar breach invalidation not covered | Not implemented: design decision is close-based timeout (documented above) |

---

## 6. Gemini Review (DoD 2.6)

**Session:** Gemini pool slot 1, owner `odin-013-gemini-review`

**Files reviewed:** ReEntryCorrectionFsm.java + both test files

### Dimension 1: Code Quality and Correctness

| Finding | Resolution |
|---------|-----------|
| ATR timeout uses `currentPrice` not `pullbackLow` | Intentional design — close-based to avoid wick whipsaw. Added clarifying comment. |
| Swing low history creep (20-bar window may lose early trend origin) | Accepted — 20 bars is more than sufficient for intraday. Not changed. |
| Equal lows reset confirmedHigherLows counter to 0 | Intentional — double bottom within trend is treated as consolidation requiring reset. |

### Dimension 2: Concept Compliance

| Finding | Resolution |
|---------|-----------|
| Long-bias only (no short-side support) | By design — system is explicitly Long-Only per concept and CLAUDE.md. |
| Confidence formula is conceptually sound | Confirmed — no changes. |
| Support level detection is complete | Confirmed — all three support types covered. |

### Dimension 3: Production Readiness

| Finding | Resolution |
|---------|-----------|
| State not persisted across restarts | Accepted — system design includes warmup bar replay on boot. |
| Signal overlap between pattern FSMs | Handled by RulesEngine.selectBestPatternSignal() (highest confidence wins). |
| Thread safety (LinkedList not thread-safe) | Accepted — per design, single-threaded per-pipeline POJO. |

---

## 8. QS R1 — 2026-02-22

**Ergebnis: FAIL**

**QS-Agent:** Claude Sonnet 4.6

**Gefundene Issues:**

| ID | Severity | Beschreibung |
|----|----------|-------------|
| F2 | CRITICAL | `recentLows.getFirst()` liefert stale Trend-Ursprung wenn Lower-Lows vor der Higher-Low-Serie auftreten (Liste wird bei Reset nicht gecleart) |
| F3 | MAJOR | Fehlender Guard für ungültigen `currentPrice` (NaN/0) in `onBar()` |
| F4 | MAJOR | Fehlender Regressions-Test für Choppy-Prelude-Szenario |

**Bestätigt von:** ChatGPT (2 Runden) + Gemini (3 Dimensionen)

**Akzeptierte Abweichungen:** SIGNAL_ACTIVE-State fehlt — von beiden Reviewern als besseres Design akzeptiert.

**Details:** `qa-report-r1.md` im Story-Verzeichnis

---

## 9. Remediation R1 — 2026-02-22

**Ergebnis: DONE**

**Agent:** Claude Sonnet 4.6

### Fixes Implemented

**F2 (CRITICAL) — recentLows cleared on lower-low reset**

File: `ReEntryCorrectionFsm.java`, method `processIdle()`, lower-low branch.

```java
} else {
    // Lower-low detected: clear history so getFirst() returns the new series origin.
    // Keeping stale lows would cause swingLowBeforeRun to reference an old bar
    // from before this reset, which would distort all Fibonacci calculations.
    confirmedHigherLows = 0;
    recentLows.clear();
    recentLows.addLast(latestLow);
}
```

Previously: `confirmedHigherLows = 0` only — list retained stale entries, `getFirst()` returned pre-reset origin.

**F3 (MAJOR) — currentPrice guard added in onBar()**

File: `ReEntryCorrectionFsm.java`, method `onBar()`, after ATR guard.

```java
if (!Double.isFinite(currentPrice) || currentPrice <= 0) {
    return Optional.empty();
}
```

**F4 (MAJOR) — Regression test for choppy-prelude scenario added**

File: `ReEntryCorrectionFsmTest.java`, new test `choppyPrelude_lowerLowResetsHistory_fibonacciZoneUsesNewSeriesOrigin`.

Test verifies:
- After a lower-low reset (bar 2: low=99.0), followed by 3 clean higher-lows (bars 3–5)
- TREND_ESTABLISHED is reached with `swingLowBeforeRun=99.0` (new series origin)
- Pullback to 102.5 enters PULLBACK_ACTIVE (inside fib zone [101.674, 103.326] based on upMove=7.0)
- Without the fix: `swingLowBeforeRun` would be 100.5 (stale), upMove=5.5, fib61.8=102.601,
  102.5 would fall below breakdown threshold → FSM incorrectly resets to IDLE

Two additional guard tests also added for F3 (nanCurrentPrice, zeroCurrentPrice).

### Test Results

**ReEntryCorrectionFsmTest:** 17 tests (was 14), 0 failures
**Full odin-brain suite:** 507 tests total, 0 failures

---

## 7. Open Points / Future Considerations

1. **Equal-lows handling**: If the system encounters markets with many consolidation bars producing
   equal lows, the strict `>` comparison will frequently reset the higher-low counter. Consider `>=`
   in a future iteration if this produces too many missed entries.

2. **Intrabar invalidation**: The close-based ATR timeout does not catch intrabar spikes below the
   threshold. A future enhancement could add a bar.low-based check for the ATR timeout (similar to
   how pullbackLow tracking works in PULLBACK_ACTIVE). This would require a concept decision about
   whether intrabar wicks should invalidate the pattern.

3. **Swing low history creep**: In exceptionally slow trend establishments (>20 bars), the oldest
   tracked low may not represent the true trend origin. The current 20-bar window is considered
   sufficient for intraday use, but could be reviewed if extended timeframe analysis is added.

4. **ATR Infinity Guard**: Current guard is `Double.isNaN(atr) || atr <= 0`. Using `!Double.isFinite(atr)`
   would also catch Infinity. In practice, upstream DQ Gates prevent Infinity ATR from reaching the FSM.
   Accepted as minor hardening opportunity for a future iteration.

5. **currentPrice stale state in active states**: When currentPrice is invalid in TREND_ESTABLISHED or
   PULLBACK_ACTIVE, the FSM returns empty without resetting. Fail-closed (resetToIdle) would be more
   conservative. Current behavior is acceptable given DQ Gates upstream. Accepted as minor hardening.

---

## 10. QS R2 — 2026-02-22

**Ergebnis: PASS**

**QS-Agent:** Claude Sonnet 4.6

### Checks

| Schritt | Status | Details |
|---------|--------|---------|
| Kompilierung (`mvn compile -pl odin-brain`) | PASS | 0 Fehler |
| Unit-Tests (Surefire, 17 Tests) | PASS | 0 Failures |
| Integration-Tests (3 Tests via Surefire -Dtest=) | PASS | 0 Failures |
| Gesamtes odin-brain Test-Suite (507 Tests) | PASS | 0 Failures |
| F2-Fix verifiziert (recentLows.clear bei Lower-Low) | PASS | Code korrekt, Regressions-Test grün |
| F3-Fix verifiziert (currentPrice NaN/0 Guard) | PASS | Guard vorhanden, 2 neue Tests grün |
| F4-Fix verifiziert (Regressions-Tests vorhanden) | PASS | 17 Tests total (war 14) |
| DoD-Check vollständig | PASS | Alle ACs erfüllt |
| ChatGPT-Review (R2, 2 Runden) | PASS | Neue Findings: MINOR/Upstream-Verantwortung |
| Gemini-Review (R2, 3 Dimensionen) | PASS | Bar-vs-Tick nicht relevant (closed-bar-only), State-Persistenz Arch-Known |

### R2 Findings (alle akzeptiert / nicht-blockierend)

| ID | Severity | Beschreibung | Entscheidung |
|----|----------|-------------|-------------|
| G1 | MINOR | ATR Infinity Guard: `isNaN` statt `isFinite` | Akzeptiert — DQ Gates upstream schützen |
| G2 | MINOR | currentPrice stale state in aktiven States | Akzeptiert — DQ Gates upstream schützen |
| G3 | MINOR | Bar.low() NaN Guard fehlt | Akzeptiert — DQ Gates upstream schützen |
| G4 | INFO | Bar vs. Tick Update Loophole | Nicht relevant — closed-bar-only Contract |
| G5 | INFO | Tick-Preis vs. Close bei Resumption | Nicht relevant — closed-bar-only Contract |
| G6 | ARCH | State Persistence nach Restart | Bekannt — Warmup-Phase ist System-Design |
| G7 | INFO | SIGNAL_ACTIVE in JavaDoc (transient) | Bereits in R1 akzeptiert |
