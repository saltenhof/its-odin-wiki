# ODIN-034 Implementation Protocol — Challenger Suite

**Story:** ODIN-034 — Backtest Challenger Suite
**Date:** 2026-02-23
**Agent:** Implementation Agent (Claude Sonnet 4.6)

---

## 1. Scope

Implementation of a 20-scenario regression and release-gate framework for the ODIN trading system.
The Challenger Suite provides a deterministic, reproducible set of representative intraday market
conditions against which strategy changes must be validated before production deployment.

---

## 2. Files Produced

### Main Sources (odin-backtest)

| File | Description |
|------|-------------|
| `de.its.odin.backtest.challenger.ChallengerScenario` | Enum: 20 constants (S01–S20) across 5 categories. Nested `ScenarioCategory` enum. `SCENARIO_COUNT = 20` constant. |
| `de.its.odin.backtest.challenger.ChallengerSuiteReport` | Immutable record with compact constructor validation. Nested `ScenarioResult` record. `failingResults()` helper. |
| `de.its.odin.backtest.challenger.ChallengerSuiteRunner` | Spring `@Component`. 3 `runSuite()` overloads. Registry with all 20 specs. Fail-fast `validateRegistryCompleteness()`. |

### Test Sources (odin-backtest)

| File | Description |
|------|-------------|
| `ChallengerScenarioTest` | Unit: 20 constants, SCENARIO_COUNT, categories, descriptions, uniqueness (9 tests) |
| `ChallengerSuiteReportTest` | Unit: construction happy paths, compact constructor invariants K2+K3, failingResults(), immutability (17 tests) |
| `ChallengerSuiteRunnerTest` | Unit: all-pass, all-fail, mixed, minPassRate gate, determinism, null guards, ScenarioSpec validation (17 tests) |
| `ChallengerSuiteRunnerIntegrationTest` | Integration (Failsafe): full pipeline end-to-end with stub BacktestRunner, 5 scenarios (5 tests) |

**Total: 48 tests across 4 test classes.**

---

## 3. Design Decisions

### D1: Pass/Fail Criterion — Minimum 1 Trade (V1)

The scenario pass criterion is: `totalTrades >= MIN_TRADES_FOR_ACTIVE_SCENARIO (= 1)`.
This intentionally simple criterion verifies that ODIN engaged with the market at all during the
scenario date. Richer per-scenario assertions (stop trigger confirmation, EOD flat verification,
LLM circuit-breaker engagement) require event-level backtest output not yet exposed by
`BacktestReport`. This is a documented V1 limitation.

**Justification:** Enforcing that the system was active across 20 diverse market days is a
meaningful first release gate. A zero-trade day means either the historical data is missing or the
strategy rejected every entry — both are legitimate failure signals.

### D2: Sequential Execution for Determinism

All 20 scenarios run in `ChallengerScenario.values()` declaration order.
**Rejected:** Parallel execution via `ExecutorService`. V1 prioritises determinism and avoidance of
DB contention. Parallelisation can be added later without API changes (the report format is identical).

### D3: Fail-Fast Registry Completeness

`validateRegistryCompleteness()` is called in the constructor. If any `ChallengerScenario` enum
constant lacks a `ScenarioSpec` in the registry, an `IllegalStateException` is thrown at startup
rather than silently producing incomplete suite results at runtime.

### D4: Record Invariants (K2 and K3)

**K2 (ChallengerSuiteReport):**
- `passedScenarios` must match the actual count of passing entries in `scenarioResults`
- `suiteVerdict` must equal `passRate >= minPassRate`

**K3 (ScenarioResult):**
- `passed == true` requires `failureReason == null`
- `passed == false` requires `failureReason` non-null and non-blank

These invariants prevent silent data corruption where the report fields are inconsistent with the
underlying result data. They are enforced in compact constructors with clear error messages.

### D5: Defensive Map Copy

`ChallengerSuiteReport` calls `Map.copyOf(scenarioResults)` in the compact constructor to prevent
callers from mutating the passed-in map after construction.

---

## 4. ChatGPT Review — Round 1

**Reviewer:** ChatGPT (Claude Code ChatGPT Pool, slot acquired 2026-02-23)

**Key Findings and Disposition:**

| ID | Finding | Disposition | Action |
|----|---------|-------------|--------|
| K1 | `Instant.now()` / UUID non-determinism (metadata path) | REJECTED | Not in trading codepath; metadata only |
| K2 | `passedScenarios` not validated against actual map count | ACCEPTED | Implemented in `ChallengerSuiteReport` compact constructor |
| K3 | `passed=false` requires non-blank `failureReason` invariant | ACCEPTED | Implemented in `ScenarioResult` compact constructor |
| W1 | Null-safety for `BacktestRunner.run()` return value | ACCEPTED | Added null check before `evaluateResult()` |
| W2 | Weak PASS/FAIL criterion | REJECTED | V1 intentionally simple; documented as limitation |
| W3 | Failure message too generic | ACCEPTED as HINWEIS | Improved message with root cause hints |
| W4 | Magic values in scenario registry | REJECTED | V1 acceptable; real data substitutable later |
| W5 | Registry completeness not validated at startup | ACCEPTED | Added `validateRegistryCompleteness()` in constructor |
| W6 | `minPassRate` should be `@ConfigurationProperties` | REJECTED | Not in story scope; no Config task ordered |
| H1 | Unused `Arrays` import | ACCEPTED | Removed; replaced with `ArrayList` |

## 5. ChatGPT Review — Round 2

**Confirmed priorities:** K2, K3, W1, W5, H1.
**Additional recommendation:** Fail-fast registry check should throw `IllegalStateException` (not `IllegalArgumentException`) because it is a programming error at wiring time, not a user input error. Implemented accordingly.

---

## 6. Gemini Review — 3 Dimensions

**Reviewer:** Gemini 2.5 Pro (Claude Code Gemini Pool, slot 0, 2026-02-23)

### Dimension 1: Code Quality

Verdict: **Excellent.** Java 21 idioms used correctly throughout (records, `.toList()`, `List.getFirst()`). No `var`, no magic numbers, no reflection. Naming conventions strict. JavaDoc complete on all public members. Test coverage comprehensive.

### Dimension 2: Concept Fidelity

Verdict: **Correct.** All 20 scenarios correctly mapped to 5 categories. Registry completeness validated at startup. K2/K3 invariants correctly enforced in compact constructors.

**Notable finding (W):** Pass criterion is a V1 simplification acknowledged in code and docs. A future V2 should assert on specific `BacktestReport` metrics (PnL direction, max drawdown bounds) to make the release gate genuinely robust. Tracking as backlog item.

**Notable finding (W):** Hardcoded historical dates in registry may fail if the underlying database purges 2023 data. Test data availability must be pinned in environments that run the Challenger Suite.

### Dimension 3: Production Readiness

Verdict: **Ready for V1.** Exception handling is graceful — mid-flight backtest exceptions produce a `failResult` rather than crashing the suite. Null checks cover the `BacktestRunner` return value. Logging uses SLF4J appropriately at INFO/WARN/ERROR levels.

**Finding (actionable — CHANGE):** Mockito `when` stubs in tests used `isNull()` for the progress callback argument. This causes stubs to stop matching if the production code changes to pass a real callback. Fixed: `when` stubs now use `any()` for the callback; `verify` calls retain `isNull()` for strict verification.

### Gemini Round 2 — Pushback Evaluation

| Question | Answer | Action |
|----------|--------|--------|
| Floating-point invariant risk (passRate vs minPassRate) | KEEP_AS_IS — both values derived from same IEEE 754 arithmetic, no divergence risk | No change |
| Literal `20` in tests vs `SCENARIO_COUNT` constant | KEEP_AS_IS — intentional tripwire; using the constant would mask genuine test failures when a 21st scenario is added | No change |
| `isNull()` in `when` vs `verify` | CHANGE — stubs use `any()`, verifies keep `isNull()` | Implemented |

---

## 7. Final Changes Summary (Post-Review)

1. Removed unused `Arrays` import in `ChallengerSuiteRunner` (replaced with `ArrayList`)
2. Added `validateRegistryCompleteness()` called from constructor (W5)
3. Added null-safety check for `report == null || report.summary() == null` (W1)
4. Improved zero-trade failure reason with root cause hints (W3)
5. Added K2 invariants in `ChallengerSuiteReport` compact constructor
6. Added K3 invariants in `ScenarioResult` compact constructor
7. Added K2+K3 invariant tests in `ChallengerSuiteReportTest`
8. Fixed Mockito stubs: `isNull()` → `any()` in `when()` calls in both test files
9. Removed unused `Consumer` and `isNull` imports from test files
10. Removed unused `SimulationRunner` import from `ChallengerSuiteRunnerTest`

---

## 8. Compilation Status

- Main sources: `BUILD SUCCESS` (no warnings in challenger package)
- Test sources: `BUILD SUCCESS` (all 7 challenger test/source files compile cleanly)
- Pre-existing test failures in `BacktestRunnerTest`, `BacktestRunnerGovernanceIntegrationTest`,
  `WalkForwardRunnerIntegrationTest` are unrelated to ODIN-034 (reference removed `CoreProperties`
  inner classes from prior story ODIN-033). Not introduced by this implementation.

---

## 9. Git Status

All 7 files staged (`git add`):

- `A odin-backtest/src/main/java/de/its/odin/backtest/challenger/ChallengerScenario.java`
- `A odin-backtest/src/main/java/de/its/odin/backtest/challenger/ChallengerSuiteReport.java`
- `A odin-backtest/src/main/java/de/its/odin/backtest/challenger/ChallengerSuiteRunner.java`
- `A odin-backtest/src/test/java/de/its/odin/backtest/challenger/ChallengerScenarioTest.java`
- `A odin-backtest/src/test/java/de/its/odin/backtest/challenger/ChallengerSuiteReportTest.java`
- `A odin-backtest/src/test/java/de/its/odin/backtest/challenger/ChallengerSuiteRunnerTest.java`
- `A odin-backtest/src/test/java/de/its/odin/backtest/challenger/ChallengerSuiteRunnerIntegrationTest.java`

**No commit performed** — per instructions, commit is the responsibility of the QS-Agent.

---

## 10. Acceptance Criteria Coverage

| Criterion | Status |
|-----------|--------|
| `ChallengerScenario` enum with exactly 20 constants S01–S20 | DONE |
| 5 categories: TREND_DAY, RANGE_DAY, VOLATILE_DAY, EDGE_CASE, STRESS | DONE |
| `ChallengerSuiteReport` immutable record with per-scenario results and suite verdict | DONE |
| `ChallengerSuiteRunner` orchestrates all 20 scenarios via `BacktestRunner` | DONE |
| Configurable `minPassRate` (default 1.0) | DONE |
| Sequential execution in enum declaration order | DONE |
| Unit tests for all three classes | DONE |
| Integration test end-to-end with stub `BacktestRunner` for ≥3 scenarios | DONE |
| ChatGPT review ≥2 rounds | DONE (2 rounds) |
| Gemini review ≥3 dimensions | DONE (3 dimensions + round 2 pushback) |
| No commit (QS-Agent responsibility) | DONE |
