# ODIN-019 Dual-Key Arbiter — Implementation Protocol

## Story Summary

Implemented the Symmetric Hybrid Protocol (Concept 08) in the Decision Arbiter:
- Entry: Both keys must agree (Quant AND LLM)
- Exit: Either key suffices (Either-Key Rule)

## New Files

| File | Package | Purpose |
|------|---------|---------|
| `VoteAction.java` | `odin-brain/arbiter` | Enum of structured vote actions (ALLOW_ENTRY, EXIT_NOW, NO_TRADE, ...) |
| `QuantVote.java` | `odin-brain/arbiter` | Record: gate-cascade + rules result as structured Quant vote |
| `LlmVote.java` | `odin-brain/arbiter` | Record: bounded LLM tactical output as structured LLM vote |
| `FusionResult.java` | `odin-brain/arbiter` | Record: immutable fusion outcome with fused regime + parameters + winner side |
| `FusionAction.java` | `odin-brain/arbiter` | Enum: final trading action from fusion (ENTER, EXIT, NO_ACTION, HOLD, ...) |
| `WinnerSide.java` | `odin-brain/arbiter` | Enum: decisive vote side for forensic logging (QUANT, LLM, BOTH, QUANT_ONLY) |
| `DecisionResult.java` | `odin-brain/arbiter` | Record: decide() return type wrapping Optional<TradeIntent> + FusionResult (always present) |
| `DecisionArbiterDualKeyTest.java` | `odin-brain/test` | Unit tests for all AC-01..AC-11 (28 tests) |
| `DecisionArbiterDualKeyIntegrationTest.java` | `odin-brain/test` | Integration tests with real GateCascadeEvaluator (6 tests) |

## Modified Files

| File | Change |
|------|--------|
| `DecisionArbiter.java` | Complete refactor: Dual-Key protocol, DecisionResult return type, Either-Key Exit, confidence minimum, explicit rank maps |
| `RulesResult.java` | Added `hasOpenPosition` boolean for LLM-triggered Either-Key Exit |
| `BrainProperties.java` | Added `minConfidenceEntry` (double, default 0.3) to LlmProperties |
| `odin-brain.properties` | Added `odin.brain.llm.min-confidence-entry=0.3` |
| `TestBrainProperties.java` | Updated LlmProperties constructor call (+minConfidenceEntry=0.3) |
| `PipelineFactory.java` | Removed QuantValidation instantiation, updated DecisionArbiter to 3-arg ctor |
| `TradingPipeline.java` | Added DecisionResult import, updated arbiter.decide() call handling |
| `DecisionArbiterTest.java` | Updated to new DecisionResult API (hasIntent(), intent().get()) |
| `DecisionArbiterIntegrationTest.java` | Rewritten for new API with LlmTacticalOutput |
| `KpiEngineTest.java` | LlmProperties ctor +minConfidenceEntry |
| `CallCadencePolicyTest.java` | LlmProperties ctor +minConfidenceEntry |
| `LlmAnalystOrchestratorTest.java` | LlmProperties ctor +minConfidenceEntry |
| `LlmTacticalPipelineIntegrationTest.java` | LlmProperties ctor +minConfidenceEntry |
| `OptimizationAnalystClientTest.java` | LlmProperties ctor +minConfidenceEntry |
| `PipelineFactoryTest.java` | Fixed stale: LlmProperties, PatternProperties, RulesProperties, DecisionResult mock |
| `TradingPipelineTest.java` | Updated arbiter mock returns from Optional to DecisionResult |

## ChatGPT Review (3 Rounds)

**Round 1 — Critical Findings Fixed:**
- P0: Either-Key Exit not implemented → Added LLM-triggered exit via `hasOpenPosition` in `fuse()`
- P0: Fusion parameters not in TradeIntent → TradeIntent stays lean; callers read from `DecisionResult.fusion()`
- P1: SAFETY_VETO dead code → Removed from VoteAction enum
- P1: ordinal() fragile → Replaced with explicit `PROFIT_PROTECTION_RANK` and `SIZE_MODIFIER_RANK` Maps
- P1: quantValidation unused → Removed from constructor and callers
- P1: LLM confidence not checked → Added `minConfidenceEntry` (0.3) with floor in `buildLlmVote()`

**Round 2 — Architecture Confirmation:**
- DecisionResult wrapper confirmed: FusionResult always present for forensics
- Regime ranking: Concept doc is authoritative (HIGH_VOL before RANGE_BOUND)

**Round 3 — Final Confirmation:**
- Change list confirmed as complete
- P0 implementation scope validated per AC-04

## Gemini Review (3 Dimensions)

**Dimension 1 — Code Quality:** No critical issues. Minor: Map.of imports were fully qualified → already acceptable but left as-is (avoids name clash with existing Map usage).

**Dimension 2 — Concept Conformity:**
- P1 NaN confidence: `NaN < 0.3` evaluates to false in Java → Added `!Double.isFinite()` guard
- P2 Confidence boundary: `< minConfidence` allows exactly 0.3 (inclusive lower bound) — confirmed intentional

**Dimension 3 — Production Readiness:**
- P0 JSON escaping: `escapeJson()` only handled `\\` and `"` → Rewrote with switch to handle all RFC 8259 control chars (`\n`, `\r`, `\t`, `\b`, `\f`, `\uXXXX` for codepoint < 32)
- P1 Null checks: Added `Objects.requireNonNull()` to constructor and `decide()` for all mandatory params
- P2 WinnerSide for exits: Hard-coded QUANT in `fuseExit()` even for LLM-triggered exits → Added `winner` param to `fuseExit()`; caller passes `WinnerSide.LLM` for Either-Key path

## Test Count

| Module | Before | After |
|--------|--------|-------|
| odin-brain | ~650 | 659 |
| odin-core | 98 | 98 |

All tests green.

## Acceptance Criteria Coverage

| AC | Test | Status |
|----|------|--------|
| AC-01 ENTER+ENTER → ENTER | `ac01_quantEnter_llmEnter_producesEntry` | PASS |
| AC-02 ENTER+NO_TRADE → NO_ACTION | `ac02_quantEnter_llmNoTrade_producesNoAction` | PASS |
| AC-03 Gate fail → NO_ACTION | `ac03_quantGateFail_llmEnter_producesNoAction` | PASS |
| AC-04a Quant EXIT → EXIT | `ac04_quantExit_anyllmVote_producesExit` | PASS |
| AC-04b LLM EXIT + position open → EXIT | `eitherKeyExit_llmExit_positionOpen_producesExit` | PASS |
| AC-04c LLM EXIT + flat → ignored | `eitherKeyExit_llmExit_positionFlat_noExit` | PASS |
| AC-05 Hold → NO_ACTION | `ac05_noExitSignal_noEntrySignal_producesNoAction` | PASS |
| AC-06 PP fusion | `ac06_paramFusion_*` | PASS |
| AC-08 PP conservatism | `conservatismRanking_profitProtection_*` | PASS |
| AC-09 Regime fusion | `ac09_regimeFusion_*` | PASS |
| AC-10 EventLog logging | `ac10_fusionDecision_loggedToEventLog` | PASS |
| AC-11 QUANT_ONLY | `ac11_quantOnlyMode_*` | PASS |
| NEW: DecisionResult always present | `decisionResult_fusionAlwaysPresent_evenOnNoAction` | PASS |
| NEW: LLM confidence minimum | `llmLowConfidence_belowMinimum_treatedAsNoTrade` | PASS |
| NEW: WinnerSide forensics | `eitherKeyExit_llmExit_positionOpen_winnerSideIsLlm` | PASS |
| NEW: NaN defence | `llmNanConfidence_rejectedByLlmTacticalOutputConstructor` | PASS |
