# Protokoll: ODIN-073 — Pre-Trade Hard Controls

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### Pre-Trade Sizing Estimate
The PreTradeControlGate receives an estimated share count (derived from a simplified max-risk calculation) rather than the final PositionSizer output. This is by design: pre-trade controls are cheap fast-fail safety nets that run BEFORE the more expensive position sizing calculations. Using a rough estimate that may slightly overestimate the final share count is intentionally conservative — it means pre-trade controls reject more aggressively, never less. The actual share count from PositionSizer will always be <= the estimate.

### Backwards-Compatible RiskGate API
The RiskGate class retains the old 4-argument evaluate(intent, riskState, snapshot, currentAtr) signature as a backwards-compatible convenience method that delegates to the new 6-argument version with currentPositionShares=0 and avgDailyVolume=0. This avoids breaking existing test code and callers while enabling the new pre-trade controls.

### OrderRateTracker Thread Safety
ConcurrentLinkedDeque provides sufficient thread safety because each pipeline's OrderRateTracker is accessed by the pipeline's single-threaded decision loop. The ConcurrentLinkedDeque is used defensively for correctness, not because multiple threads concurrently access the same tracker. The check-then-act pattern (isWithinLimit + recordOrder) is atomic within the single-threaded pipeline loop.

### Price Collar Graceful Skip
When VWAP or ATR is unavailable (NaN, 0, negative, infinite), the price collar check is skipped rather than rejecting. This prevents false rejections during early session startup when indicators are warming up. The RiskGate's other checks (R/R ratio, exposure limits) provide protection during these periods.

### ADV Check Skip When No ADV Data
When avgDailyVolume is 0 or negative, the ADV-relative order size check is skipped. Only the absolute maxOrderShares check applies. This is per the story notes: "KEIN Blocker wenn ADV nicht verfuegbar."

## Offene Punkte

- ADV data is not currently provided to the RiskGate by the TradingPipeline (passed as 0L). When ADV data becomes available via InstrumentProfile or historical data, the pipeline should pass the real value.
- Pre-market/after-hours price collar protection is effectively disabled when VWAP/ATR are unavailable. This is acceptable for V1 but should be revisited if pre-market trading becomes more active.
- Corporate action scenarios (stock splits) may cause false ADV rejections. Not a V1 concern but noted for future awareness.

## ChatGPT-Sparring

### Session Summary
Discussed edge cases and test gaps with ChatGPT. Key findings:

**Implemented from ChatGPT suggestions:**
1. Rejected orders should NOT increment rate tracker — added 2 tests verifying this
2. Exact boundary at max order shares — added test
3. Non-finite VWAP/ATR edge cases (Inf, negative) — added 3 tests
4. Exact window boundary at 60s — added test confirming orders at T+60s are still counted
5. Zero order shares edge case — added test

**Evaluated and deferred/rejected:**
- Short position semantics: ODIN is long-only in V1, not applicable
- Concurrent rate limit atomicity: Pipeline is single-threaded, CyclicBarrier test is over-engineering
- entryPrice NaN/Inf: Already handled by RiskGate's input validation (entryPrice <= 0 check) before pre-trade controls
- Projected position including open orders: Complex, deferred to future story if needed. Current approach tracks fills only.

## Gemini-Review

### Dimension 1: Code Review
**Findings:**
1. Check-then-act race in OrderRateTracker — **Accepted, low risk:** Pipeline is single-threaded per design.
2. Estimated shares explosion with tiny stopDist — **Accepted:** Very small stop distances would produce an unrealistic share count, but this only makes the pre-trade control MORE conservative (rejects more). Not a safety gap.
3. Memory accumulation in OrderRateTracker — **Accepted, low risk:** Max 5 orders/minute, trivial memory impact.
4. Null safety in PreTradeControlGate constructor — **Accepted:** Wiring happens at pipeline creation time in PipelineFactory, which is well-tested. Adding null checks would be defensive code without practical benefit.

### Dimension 2: Concept Fidelity
All acceptance criteria verified as correctly implemented:
- Max Order Size (absolute + ADV-relative): Correct
- Price Collar (VWAP +/- N*ATR): Correct
- Order Rate Limit (sliding window): Correct
- Max Intraday Position: Correct
- Event Logging: Correct
- Check ordering: Correct
- Configuration mapping: Correct

### Dimension 3: Practical Review
**Findings:**
1. Sizing estimate mismatch — **Documented as design decision** (see above). Conservative by design.
2. Corporate actions / splits — **Noted as open point.** Not V1 concern.
3. Illiquid periods / collar bypass — **Documented.** Graceful skip is intentional.
4. Fractional shares — **Not applicable.** ODIN trades whole shares only, no fractional share support planned.
