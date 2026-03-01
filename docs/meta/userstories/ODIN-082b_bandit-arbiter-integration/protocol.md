# ODIN-082b: Bandit-Arbiter Integration -- Protocol

## Status: DONE

## Scope

Integrate the Thompson Sampling bandit (ODIN-082a) into the DecisionArbiter to provide
adaptive Quant/LLM weight allocation based on per-regime trade outcomes.

## Implementation Summary

### Production Code Changes

| File | Change |
|------|--------|
| `ThompsonSamplingBandit.java` | Added `AtomicInteger totalUpdateCount` field, `getTotalTradeCount()` method, double-clipping + re-normalization in `sampleWeights()` |
| `BrainProperties.java` | Added `AdaptiveWeightProperties` record (13th field) with `enabled`, `priorAlpha`, `priorBeta`, `minTradesForActivation`, `minWeight`, `maxWeight` |
| `FusionResult.java` | Added 17th field `adaptiveWeightsUsed`, updated `noAction()` factory (new 8-param overload), `withCounterfactual()`, `withFallbackLevel()` to propagate flag |
| `DecisionArbiter.java` | Added bandit field + 7-param constructor, `shouldUseAdaptiveWeights()` guard, updated `resolveRegimeWeights()` to accept pre-computed `adaptiveActive` flag (race-condition fix), bandit fallback chain in weight resolution, `logFusion()` payload extended |
| `PipelineFactory.java` | Added `createBanditIfEnabled()` factory method, passes bandit to DecisionArbiter 7-param constructor |
| `ParameterOverrideApplier.java` | Propagates `adaptiveWeights` through BrainProperties reconstruction |
| `odin-brain.properties` | Default config: `odin.brain.adaptive-weights.enabled=false`, prior=2.0/2.0, min-trades=20, bounds=[0.20, 0.80] |
| `TestBrainProperties.java` | Added `AdaptiveWeightProperties` with defaults to `createDefault()` |

### Test Files

| File | Type | Tests |
|------|------|-------|
| `DecisionArbiterBanditTest.java` | Unit (Surefire) | 9 tests: bandit null, disabled config, below/at/above minTrades, safety bounds, quant-only mode, trade count increment, NO_ACTION propagation |
| `DecisionArbiterBanditIntegrationTest.java` | Integration (Failsafe) | 8 tests: entry fusion adaptive, 100-decision safety bounds stress test, null fallback, below-min fallback, exit path propagation, NO_ACTION propagation, FusionResult immutability, cross-regime trade count |

### Compatibility Updates (BrainProperties 13th field)

14 existing test files updated to pass `adaptiveWeights` parameter to `new BrainProperties(...)`:
PipelineFactoryTest, ParameterOverrideApplierTest, WalkForwardRunnerIntegrationTest,
BacktestRunnerGovernanceIntegrationTest, BacktestRunnerTest, BacktestPipelineIntegrationTest,
KpiEngineTest (2x), VpinCalculatorIntegrationTest, GateCascadeEvaluatorIntegrationTest,
VpinFlagTest, VpinGateIntegrationTest (2x), CounterfactualDecisionTest,
CounterfactualDecisionIntegrationTest.

## Test Results

- Unit tests: 1520 passed, 0 failed (odin-brain)
- Integration tests: 8 passed, 0 failed (DecisionArbiterBanditIntegrationTest)
- odin-core, odin-app: all tests passed
- Full build: `mvn clean install -DskipTests` -- SUCCESS

## Review Findings

### ChatGPT Edge-Case Review

| Finding | Severity | Resolution |
|---------|----------|------------|
| Double evaluation of `shouldUseAdaptiveWeights()` in `fuse()` + `resolveRegimeWeights()` causes race condition | HIGH | FIXED: Compute `adaptiveActive` once in `fuse()`, pass as parameter to `resolveRegimeWeights()` |
| LLM weight can exceed maxWeight when bounds have slight asymmetry within SYMMETRY_TOLERANCE | MEDIUM | FIXED: Double-clip both weights + re-normalize to guarantee both stay in [minWeight, maxWeight] |
| `setDistribution()` does not adjust `totalUpdateCount` (ODIN-082c path) | LOW | DEFERRED to ODIN-082c (persistence story) -- noted in story |
| Activation is global but sampling is per-regime (cold-start by regime) | LOW | ACCEPTED: Prior Beta(2,2) provides reasonable initial exploration for unseen regimes |
| Fallback chain uses 50/50 not 60/40 as stated | INFO | Corrected: 50/50 is intentional as neutral fallback; documentation reference was inaccurate |
| `RegimeWeightProfile` may not handle future enum values | LOW | ACCEPTED: `Map.of` will return null for missing key, caught by existing fallback to 50/50 |

### Gemini 3-Dimension Review

| Dimension | Result |
|-----------|--------|
| Code Review | All positive. Double-clipping acknowledged as defensive programming (LOW). Race condition fix confirmed correct. Thread safety via ConcurrentHashMap + AtomicInteger + ThreadLocalRandom confirmed. |
| Concept Fidelity | Bayesian posterior update correct. Weight normalization (continuous allocation variant) mathematically sound. Safety bounds preserve exploration. Activation threshold is good heuristic. |
| Production Readiness | Configuration externalized and defaulted to disabled. Fallback chain resilient. Observability via adaptiveWeightsUsed flag in EventLog confirmed. PipelineFactory null-safe. |

## Design Decisions

1. **Feature disabled by default** (`enabled=false`): Conservative rollout, activated per-pipeline via config
2. **Global activation threshold** (not per-regime): Simpler, prevents edge cases with partial activation
3. **Beta(2,2) prior**: Uninformative but not uniform, provides slight regularization
4. **Symmetric safety bounds** enforced: `minWeight + maxWeight == 1.0` within tolerance
5. **Single evaluation per decision cycle**: `adaptiveActive` computed once and passed through to prevent race conditions
