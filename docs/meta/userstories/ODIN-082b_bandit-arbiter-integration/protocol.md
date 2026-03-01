# ODIN-082b: Bandit-Arbiter Integration -- Protocol

## Status: DONE (Remediation R2)

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (9 Tests: `DecisionArbiterBanditTest`)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (8 Tests: `DecisionArbiterBanditIntegrationTest`)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet (Race-Condition-Fix, Double-Clip + Re-Normalisierung)
- [x] Namespace korrigiert zu `odin.brain.arbiter.adaptive-weights.*` (Remediation R2)
- [x] Working State + Offene Punkte in protocol.md ergaenzt (Remediation R2)
- [x] Commit & Push

## Scope

Integrate the Thompson Sampling bandit (ODIN-082a) into the DecisionArbiter to provide
adaptive Quant/LLM weight allocation based on per-regime trade outcomes.

## Implementation Summary

### Production Code Changes

| File | Change |
|------|--------|
| `ThompsonSamplingBandit.java` | Added `AtomicInteger totalUpdateCount` field, `getTotalTradeCount()` method, double-clipping + re-normalization in `sampleWeights()` |
| `BrainProperties.java` | Added `AdaptiveWeightProperties` as nested record inside `ArbiterProperties` (namespace: `odin.brain.arbiter.adaptive-weights.*`). Remediation R2: moved from top-level field to sub-record of `ArbiterProperties`. |
| `FusionResult.java` | Added 17th field `adaptiveWeightsUsed`, updated `noAction()` factory (new 8-param overload), `withCounterfactual()`, `withFallbackLevel()` to propagate flag |
| `DecisionArbiter.java` | Added bandit field + 7-param constructor, `shouldUseAdaptiveWeights()` guard, updated `resolveRegimeWeights()` to accept pre-computed `adaptiveActive` flag (race-condition fix), bandit fallback chain in weight resolution, `logFusion()` payload extended. Accesses config via `properties.arbiter().adaptiveWeights()`. |
| `PipelineFactory.java` | Added `createBanditIfEnabled()` factory method, passes bandit to DecisionArbiter 7-param constructor. Uses `brainProps.arbiter().adaptiveWeights()`. |
| `ParameterOverrideApplier.java` | `BrainProperties` reconstruction updated to 12-arg constructor (adaptiveWeights now inside arbiter). |
| `odin-brain.properties` | Default config: `odin.brain.arbiter.adaptive-weights.enabled=false`, prior=2.0/2.0, min-trades=20, bounds=[0.20, 0.80] |
| `TestBrainProperties.java` | `AdaptiveWeightProperties` passed as second arg to `ArbiterProperties` constructor. |

### Test Files

| File | Type | Tests |
|------|------|-------|
| `DecisionArbiterBanditTest.java` | Unit (Surefire) | 9 tests: bandit null, disabled config, below/at/above minTrades, safety bounds, quant-only mode, trade count increment, NO_ACTION propagation |
| `DecisionArbiterBanditIntegrationTest.java` | Integration (Failsafe) | 8 tests: entry fusion adaptive, 100-decision safety bounds stress test, null fallback, below-min fallback, exit path propagation, NO_ACTION propagation, FusionResult immutability, cross-regime trade count |

### Compatibility Updates (BrainProperties restructuring, Remediation R2)

14 existing test files updated to use new `ArbiterProperties(regimeWeights, adaptiveWeights)` constructor:
PipelineFactoryTest, ParameterOverrideApplierTest, WalkForwardRunnerIntegrationTest,
BacktestRunnerGovernanceIntegrationTest, BacktestRunnerTest, BacktestPipelineIntegrationTest,
KpiEngineTest (2x), VpinCalculatorIntegrationTest, GateCascadeEvaluatorIntegrationTest,
VpinFlagTest, VpinGateIntegrationTest (2x), CounterfactualDecisionTest,
CounterfactualDecisionIntegrationTest.

## Test Results

- Unit tests: 1520 passed, 0 failed (odin-brain)
- Integration tests (ODIN-082b scope): 8 passed, 0 failed (DecisionArbiterBanditIntegrationTest)
- odin-core, odin-app, odin-backtest: 305 tests passed, 0 failed
- Full build: `mvn clean install -DskipTests` -- SUCCESS
- Note: 5 pre-existing integration test failures exist in ExhaustionDetectorIntegrationTest and pattern FSM tests (unrelated to ODIN-082b; not introduced by this story)

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
6. **Namespace `odin.brain.arbiter.adaptive-weights.*`** (Remediation R2): `AdaptiveWeightProperties` is nested as a field inside `ArbiterProperties` (not at the top level of `BrainProperties`). Rationale: adaptive weighting is a sub-concern of the arbiter's weight-resolution strategy. The `arbiter` namespace already owns `regime-weights`; `adaptive-weights` is logically its sibling as an alternative weight-resolution mechanism. This matches the story specification exactly.

## Offene Punkte

- `setDistribution()` in `ThompsonSamplingBandit` does not adjust `totalUpdateCount` when Beta-parameter state is restored from persistence. This is a known limitation tracked in ODIN-082c (Bandit Persistence). No action required for ODIN-082b.
- Multi-Pipeline: With the current implementation, each `PipelineFactory` instance creates one shared bandit for all pipelines it manages. If the system scales to multiple `PipelineFactory` instances (e.g., per-instrument), each will have its own bandit. This is the intended design for V1 (global-per-factory). Per-instrument bandits are a ODIN-082c+ consideration.
