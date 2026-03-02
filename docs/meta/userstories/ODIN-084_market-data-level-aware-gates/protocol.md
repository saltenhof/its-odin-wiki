# ODIN-084: Market Data Level-Aware Gates — Implementation Protocol

## Status
**COMPLETE** | 2026-03-01

## Scope
Make the 8-gate entry cascade data-quality-aware by introducing a `MarketDataLevel` concept.
Gates that require higher data levels are hard-skipped at lower levels, NaN artifacts of
known data limitations produce PASS with warning (graceful degradation) instead of blocking entry.

## Modules Touched
- **odin-api**: MarketDataLevel enum, GateEvaluation (skipped field), GateCascadeResult (skippedGates/skippedCount)
- **odin-brain**: EntryGate interface (two-method protocol), GateCascadeEvaluator (level-aware evaluation),
  VolumeGate (strictMode), SpreadGate (L1 requirement), OfiGate (L2 requirement)
- **odin-data**: DataProperties (marketDataLevel field), odin-data.properties (default)
- **odin-core**: PipelineFactory (passes marketDataLevel to GateCascadeEvaluator)

## Gate-Level Matrix (as implemented)

| Gate         | minimumLevelToEvaluate    | minimumLevelForStrictEnforcement | Behavior at L0           |
|--------------|---------------------------|----------------------------------|--------------------------|
| SpreadGate   | L1_QUOTES                 | L1_QUOTES                        | Hard SKIP                |
| OfiGate      | L2_DEPTH                  | L2_DEPTH                         | Hard SKIP                |
| VolumeGate   | L0_NO_EXTENDED_VOLUME     | L0_FULL_VOLUME                   | Graceful degradation     |
| RsiGate      | L0_NO_EXTENDED_VOLUME     | L0_NO_EXTENDED_VOLUME            | Always strict            |
| EmaTrendGate | L0_NO_EXTENDED_VOLUME     | L0_NO_EXTENDED_VOLUME            | Always strict            |
| VwapGate     | L0_NO_EXTENDED_VOLUME     | L0_NO_EXTENDED_VOLUME            | Always strict            |
| AtrGate      | L0_NO_EXTENDED_VOLUME     | L0_NO_EXTENDED_VOLUME            | Always strict            |
| AdxGate      | L0_NO_EXTENDED_VOLUME     | L0_NO_EXTENDED_VOLUME            | Always strict            |

## Key Design Decisions

1. **SKIPPED = PASS for cascade**: Skipped gates do not block entry. They are tracked separately
   via `skippedGates` / `skippedCount` for observability.

2. **VolumeGate strictMode**: Mutable field set by GateCascadeEvaluator before each evaluation.
   At L0_NO_EXTENDED_VOLUME: NaN volumeRatio produces PASS with warning (known artifact).
   At L0_FULL_VOLUME or above: NaN produces FAIL (genuine data error).

3. **Backward compatibility**: 1-arg GateCascadeEvaluator constructor defaults to L0_NO_EXTENDED_VOLUME.
   5-arg GateEvaluation constructor defaults skipped=false. 3-arg GateCascadeResult constructor
   defaults skippedGates=empty, skippedCount=0.

4. **Startup invariant validation**: GateCascadeEvaluator validates at construction time that
   every gate satisfies `minimumLevelForStrictEnforcement >= minimumLevelToEvaluate`.

## Files Created
- `odin-api/src/main/java/de/its/odin/api/model/MarketDataLevel.java`
- `odin-api/src/test/java/de/its/odin/api/model/MarketDataLevelTest.java`

## Files Modified

### Production Code
- `odin-brain/src/main/java/de/its/odin/brain/quant/gates/EntryGate.java` (two default methods)
- `odin-brain/src/main/java/de/its/odin/brain/quant/gates/VolumeGate.java` (strictMode, level overrides)
- `odin-brain/src/main/java/de/its/odin/brain/quant/gates/SpreadGate.java` (L1 level overrides)
- `odin-brain/src/main/java/de/its/odin/brain/quant/gates/OfiGate.java` (L2 level overrides)
- `odin-brain/src/main/java/de/its/odin/brain/quant/GateCascadeEvaluator.java` (level-aware evaluation)
- `odin-api/src/main/java/de/its/odin/api/dto/GateEvaluation.java` (skipped field)
- `odin-api/src/main/java/de/its/odin/api/dto/GateCascadeResult.java` (skippedGates/skippedCount)
- `odin-data/src/main/java/de/its/odin/data/config/DataProperties.java` (marketDataLevel field)
- `odin-data/src/main/resources/odin-data.properties` (default property)
- `odin-core/src/main/java/de/its/odin/core/pipeline/PipelineFactory.java` (passes level)

### Test Code (updated for MarketDataLevel.L2_DEPTH in pre-existing tests)
- `odin-api/src/test/java/de/its/odin/api/dto/GateCascadeResultTest.java` (+8 ODIN-084 tests)
- `odin-api/src/test/java/de/its/odin/api/dto/GateCascadeIntegrationTest.java` (+2 ODIN-084 tests)
- `odin-brain/src/test/java/de/its/odin/brain/quant/GateCascadeEvaluatorTest.java` (+8 ODIN-084 tests)
- `odin-brain/src/test/java/de/its/odin/brain/quant/GateCascadeEvaluatorIntegrationTest.java` (+4 ODIN-084 tests)
- `odin-brain/src/test/java/de/its/odin/brain/quant/gates/VolumeGateTest.java` (+5 ODIN-084 tests)
- 13 additional test files updated for L2_DEPTH constructor parameter

## Test Results
- **Total tests**: 3,823 (all passing)
- **New ODIN-084 tests**: 44 (17 MarketDataLevel + 8 GateCascadeResult + 2 GateCascadeIntegration
  + 8 GateCascadeEvaluator + 5 VolumeGate + 4 GateCascadeEvaluatorIntegration)
- **Build**: SUCCESS across all 11 modules

## LLM Reviews

### ChatGPT (owner: ODIN-084-worker)
- Architecture: Sound. Gap noted: strict/graceful only implemented for VolumeGate.
- Bugs: OfiGate NaN handling inconsistent with strict-at-L2 declaration (by design: feature flag + warmup).
  JavaDoc "7-gate" → "8-gate" (fixed). Gate numbering inconsistency (fixed).
- Trading risk: Skipping spread/OFI increases microstructure exposure. Backtest inflation risk.
- Improvements adopted: Startup invariant validation, EntryGate JavaDoc fix, gate numbering fix.

### Gemini Review 1: Code Bug Hunt (owner: ODIN-084-worker)
- Concurrency: Mutable strictMode is a risk for multi-threaded use. Noted for future
  (current system is single-threaded per instrument pipeline).
- OFI dead code: NaN pass-through at L2 is by design (feature flag + warmup tolerance).
- skippedCount incomplete after short-circuit: By design (later gates are not evaluated at all).

### Gemini Review 2: Concept Fidelity (owner: ODIN-084-worker)
- Implementation rated "highly faithful" to design concept.
- All gate-level matrix entries verified correct.
- Backward compatibility confirmed.
- Invariant enforcement confirmed.

### Gemini Review 3: Practical Trading Gaps (owner: ODIN-084-worker)
- Trading without spread at L0: Recommend pessimistic spread assumption (future story).
- OFI skip acceptable for large-cap only: Recommend universe filter (future story).
- NaN volume pass: Recommend absolute volume floor (future story).
- Monitoring: Recommend alert on level-switch and skipped-gate telemetry (future story).
- Backtest: Ensure same MarketDataLevel in backtest and live (future story).

## Follow-Up Items (future stories, NOT ODIN-084 scope)
1. Position sizing reduction when skippedCount > 0
2. Universe filter by market cap / ADV at degraded data levels
3. Absolute volume floor for VolumeGate graceful degradation
4. Telemetry counters for skipped gates per instrument
5. Thread-safety for VolumeGate strictMode if multi-threaded execution is introduced
