# ODIN-082c: Bandit Persistence and Reward -- Protocol

## Status: DONE

## Working State

- [x] Flyway migration V032 (bandit_state table with CHECK constraints)
- [x] JPA Entity BanditStateEntity (DDD: in odin-brain domain module)
- [x] JPA Repository BanditStateRepository (upsert via ON CONFLICT)
- [x] BanditStateService (POJO mediator: loadState + persistUpdate + persistBothArms)
- [x] BanditRewardCallback functional interface
- [x] ThompsonSamplingBandit: public getDistribution/setDistribution, getPriorAlpha/Beta, setTotalTradeCount
- [x] PipelineFactory: shared bandit, BanditStateService, reward callback integration
- [x] TradingPipeline: positionEntryRegime tracking, callback on trade close
- [x] Unit tests: BanditStateServiceTest (22 tests)
- [x] Integration tests: BanditStateServiceIntegrationTest (6 tests, Zonky embedded Postgres)
- [x] DB tests: BanditStateRepositoryIntegrationTest (7 tests, Zonky embedded Postgres)
- [x] ChatGPT sparring for edge cases
- [x] Gemini review (3 dimensions: code, concept, practical)
- [x] Review findings addressed
- [x] Commit & Push

## Scope

Add database persistence for the Thompson Sampling bandit's Beta distribution parameters
and integrate post-trade reward feedback so the bandit can learn from trade outcomes across
system restarts.

## Implementation Summary

### Production Code Changes

| File | Change |
|------|--------|
| `V032__create_bandit_state.sql` | Flyway migration: creates `odin.bandit_state` table (max 10 rows: 5 regimes x 2 arms). CHECK constraints on alpha > 0, beta > 0, trade_count >= 0 |
| `BanditStateEntity.java` | JPA entity in `de.its.odin.brain.persistence`. Immutable after construction (no setters), equals/hashCode on id |
| `BanditStateRepository.java` | JPA repository: `findByRegimeAndArm()`, `upsertState()` with `INSERT ... ON CONFLICT DO UPDATE` |
| `BanditStateService.java` | POJO mediator: `loadState()` restores distributions + totalTradeCount from DB. `persistUpdate()` and `persistBothArms()` write state via TransactionTemplate. Per-row error handling, graceful degradation on DB failures |
| `BanditRewardCallback.java` | `@FunctionalInterface` for decoupled reward signaling from TradingPipeline to bandit |
| `ThompsonSamplingBandit.java` | Made `getDistribution()`, `setDistribution()` public. Added `getPriorAlpha()`, `getPriorBeta()`, `setTotalTradeCount()` |
| `PipelineFactory.java` | Shared bandit at factory level, BanditStateService created and loadState called at construction. `createBanditRewardCallback()` updates both arms + persists in single TX. Trade count derived from `alpha + beta - priorAlpha - priorBeta` |
| `TradingPipeline.java` | Added `BanditRewardCallback` field, `positionEntryRegime` tracking. On position close: calls `banditRewardCallback.onTradeCompleted(positionEntryRegime, profitable)` |
| `CoreConfiguration.java` | Passes `BanditStateRepository` and `TransactionTemplate` to PipelineFactory |
| `BacktestRunner.java` | Null params for repository/transactionTemplate (backtest does not persist bandit state) |

### Test Files

| File | Type | Tests |
|------|------|-------|
| `BanditStateServiceTest.java` | Unit (Surefire) | 22 tests: loadState empty/with entries/DB error/invalid regime/invalid arm/non-positive alpha, totalTradeCount restoration, persistUpdate correct values/DB error/uses clock, persistBothArms correct/DB error/null checks, constructor null checks, persistUpdate null checks |
| `BanditStateRepositoryIntegrationTest.java` | Integration (Failsafe) | 7 tests: migration V032, save+find roundtrip, findAll, upsert new, upsert existing (no duplicate), full roundtrip |
| `BanditStateServiceIntegrationTest.java` | Integration (Failsafe) | 6 tests: persist+load roundtrip, multiple entries, upsert same pair, empty DB, totalTradeCount restoration, persistBothArms roundtrip |

### Compatibility Updates

| File | Change |
|------|--------|
| `PipelineFactoryTest.java` | Added null params for new constructor |
| `TradingPipelineTest.java` | Added null param for BanditRewardCallback |
| `DegradationManagerIntegrationTest.java` | Added null param for BanditRewardCallback |

## Test Results

- Unit tests (odin-brain): 22 passed, 0 failed (BanditStateServiceTest)
- Integration tests (odin-brain): 13 passed, 0 failed (6 service + 7 repository)
- odin-core: 321 tests passed, 0 failed
- odin-backtest: 369 tests passed, 0 failed
- Full build: `mvn clean install -DskipTests` -- SUCCESS

## Review Findings

### ChatGPT Edge-Case Review

| Finding | Severity | Resolution |
|---------|----------|------------|
| Trade count derivation wrong: used hardcoded `-2` instead of `priorAlpha + priorBeta` | P0 | FIXED: Now uses `alpha + beta - priorAlpha - priorBeta` via `bandit.getPriorAlpha() + bandit.getPriorBeta()` |
| totalUpdateCount not restored on restart (arbiter activation resets) | P0 | FIXED: `loadState()` sums all per-(regime,arm) trade counts and calls `bandit.setTotalTradeCount()` |
| Both arms get identical reward (no differential learning) | MEDIUM | ACCEPTED by design: dual-key decision means both arms participated. Differential reward requires counterfactual analysis (ODIN-071 scope) |
| Partial persistence across arms (QUANT persisted, LLM fails) | MEDIUM | FIXED: New `persistBothArms()` writes both arms in single TransactionTemplate block |
| One bad row in loadState can break all restores | MEDIUM | FIXED: Per-row try-catch, invalid rows skipped with warning, valid rows still restored |
| DB can store NaN/negative alpha/beta | LOW | FIXED: Added CHECK constraints in migration + validation in loadState |
| Logging loses stack traces | LOW | FIXED: Exception object passed to LOG.warn |
| Concurrent persistence regression | LOW | ACCEPTED: Monotonic upsert via WHERE clause deferred. Current "last write wins" is acceptable for low-frequency trade completions |
| Transaction ownership ambiguity (TransactionTemplate + @Transactional) | INFO | ACCEPTED: @Transactional on repository joins existing TX from TransactionTemplate; creates own TX when called directly from tests |

### Gemini 3-Dimension Review

| Dimension | Result |
|-----------|--------|
| Code Review | totalTradeCount double-counting noted (2 updateReward calls per trade = counter increments by 2). This is consistent: DB restore sums per-arm counts which match. minTradesForActivation must be calibrated in "update calls" not "trades". |
| Concept Fidelity | Identical reward signals to both arms acknowledged as limitation. System learns regime-level profitability patterns, not differential arm preference. Full differential learning deferred to ODIN-071 counterfactual integration. |
| Production Readiness | Graceful degradation confirmed. Cross-instrument learning is by design (regime x arm, not instrument x regime x arm). Suggested adding metrics/telemetry for DB failure monitoring -- noted as future improvement. |

## Design Decisions

1. **Shared bandit at factory level** (not per-pipeline): The bandit learns across instruments, keyed by (Regime, BanditArm). Per-instrument learning would require N x 10 distributions and significantly more data.
2. **Both arms get same reward signal**: Both QUANT and LLM participate in every dual-key decision. Differential credit assignment requires counterfactual analysis (ODIN-071) which is out of scope.
3. **POJO, not Spring bean**: BanditStateService uses TransactionTemplate for explicit TX management, consistent with other per-pipeline POJOs.
4. **Graceful degradation**: DB failures do not crash the system. In-memory state remains authoritative. Persist retries on next trade.
5. **V032 migration** (story said V030): V030/V031 already existed, used next available version number.
6. **Backtest does not persist bandit state**: BacktestRunner passes null repository/transactionTemplate. Backtest bandit learning is ephemeral.
7. **totalTradeCount restoration**: Sum of all per-(regime,arm) trade counts from DB, set via new `setTotalTradeCount()` method. Ensures arbiter activation threshold survives restarts.

## Offene Punkte

- **Differential reward learning**: Both arms receive identical reward signals, so the bandit currently learns regime-level profitability patterns but cannot differentiate arm contributions. Full differential learning via counterfactual analysis is tracked in ODIN-071.
- **Monitoring/Telemetry**: No metric emission (Prometheus counter) for DB failures during bandit persistence. Failures are logged but not surfaced to monitoring dashboards. Consider adding DiagnosticSink integration in a future iteration.
- **Concurrent persistence**: Under high-frequency concurrent trade completions, "last write wins" upsert could theoretically regress DB state. Mitigated by low trade frequency in practice. A monotonic upsert guard (`WHERE trade_count <= EXCLUDED.trade_count`) could be added if needed.
