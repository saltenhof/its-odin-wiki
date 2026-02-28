# ODIN-064 PriceActionTouchMap -- Implementation Protocol

## Working State

| Step | Status | Notes |
|------|--------|-------|
| Read sources | Done | story.md, 07-sparring, 08-research, CLAUDE.md |
| Implement PriceActionTouchMap | Done | 315 lines, all P0 findings addressed |
| Implement SrLevelEngine integration | Done | onSnapshot step 4a, promoteFromPriceActionMap, isClusterOnly |
| Add SrLevelSource.PRICE_ACTION | Done | Prior 0.55, category "PRICE_ACTION" |
| Add SrProperties.PriceActionProperties | Done | 8 parameters, @Validated with @DecimalMin/@DecimalMax |
| Add properties defaults | Done | odin-brain.properties |
| Update all SrProperties consumers | Done | 12+ files across 4 modules |
| Unit tests (PriceActionTouchMapTest) | Done | 16 tests, all green |
| Integration tests (PriceActionTouchMapIntegrationTest) | Done | 7 tests, all green |
| All odin-brain tests | Done | 1098 tests, 0 failures |
| Clean install | Done | odin-brain, odin-core, odin-app, odin-backtest all compile |
| ChatGPT review | Done | 2 rounds, P0 findings addressed |
| Gemini review | Done | 3 dimensions covered |

## Design Decisions

### 1. Body-Position Threshold for Bucket Seeding (P0.2 fix)
New buckets are only created when the bar's close demonstrates a genuine rejection:
- Support: close must be in the upper 50% of bar range (bodyPosition >= 0.50)
- Resistance: close must be in the lower 50% of bar range
This prevents breakout/breakdown candles and ordinary wick noise from seeding spurious buckets.

### 2. Scan-Range Filtering in getCandidates() (P0.1 fix)
`getCandidates()` stores `lastCurrentPrice` from the most recent `onBar()` call and uses it
to filter candidates to within `scanRangeAtrFactor * atr5m` of the current price. This prevents
stale far-away levels from being promoted.

### 3. isFinite Guards (P0.3 fix)
All `Double.isNaN` checks were replaced with `Double.isFinite` to also catch `Infinity`.
Added `currentPrice` finite+positive guard in `onBar()`.

### 4. Decay Factor Clamped to [0, 1] (P1 fix)
Runtime clamp: `Math.min(1.0, Math.max(0.0, config.recencyDecayPerBar()))`.
Also added `@DecimalMax("1.0")` on `SrProperties.PriceActionProperties.recencyDecayPerBar`.

### 5. ATR Bin-Key Drift (Acknowledged Trade-off)
When ATR changes during the session, the same price may map to different bin keys.
This is a known trade-off: freezing ATR conflicts with ODIN's "ATR is always live" principle.
Mitigation: fusionRadius in SrLevelEngine prevents close duplicates from being promoted.

## ChatGPT Sparring

### Round 1 -- Full Review
**Findings:**
- P0.1: getCandidates() JavaDoc promises scan-range filtering but code doesn't filter -- FIXED
- P0.2: New-bucket creation predicate too weak (any non-flat bar creates a bucket) -- FIXED
- P0.3: Missing Infinity/non-finite guards -- FIXED
- P1.4: Age semantics could be misinterpreted -- FIXED (JavaDoc clarified)
- P1.5: ATR bin-key drift -- ACKNOWLEDGED (known trade-off)
- P1.6: Config validation needs @DecimalMax for decay -- FIXED
- NMS comparator reversal in suppressOverdenseZones() -- OUT OF SCOPE (pre-existing)
- PRICE_ACTION level pruning for long sessions -- OUT OF SCOPE (follow-up)

### Round 2 -- Verification
- All P0 fixes confirmed adequate
- Remaining P1 items addressed: getCandidates returns List.of() when no price ref, decay clamped
- Multi-counting when rejection bands overlap buckets: ACKNOWLEDGED (low risk with current params)

## Gemini Review -- Three Dimensions

### Dimension 1: Code Review
- Integer overflow at binning: theoretical risk for extreme prices, not relevant for ODIN's equity universe
- Memory management: correctly bounded by maxBuckets eviction
- Null/state safety: robust guards confirmed
- Decay/age off-by-one: logic sound
- Body-position threshold: correct implementation

### Dimension 2: Concept Fidelity
- Rejection definition: matches spec, enhanced with body-position threshold
- Promotion logic and fusion radius: implemented exactly as designed
- NMS integration: isClusterOnly correctly updated
- ATR-relative parametrization: all spatial variables scale with atr5m

### Dimension 3: Practice Review
- ATR bin-key drift: critical risk identified, acknowledged as known trade-off
- Pre-market bars: PriceActionTouchMap processes all bars unconditionally, but body-position threshold mitigates noise
- Trading halts: bar-based decay is intentional per spec (not time-based)
- Wide spreads: close-based signal can be noisy in illiquid stocks, outside ODIN-064 scope

## Anti-Overfitting Gedankenexperimente

### "Would this work for a pharma stock with gap-up and consolidation?"
Yes. ATR-scaled buckets adapt to the instrument's volatility. A gap-up creates a new price level;
if price consolidates and repeatedly bounces off a support level post-gap, the map will detect it.
The gap itself creates a single bucket that decays without additional rejections.

### "Would this work for an ETF like QQQ with tight spreads?"
Yes. QQQ's low ATR ($0.30-0.50) creates tight buckets ($0.03-0.05). The tighter the buckets,
the more precise the level detection. ETFs' clean price action benefits the body-position filter.

### "What happens for a stock that only falls all day (no bounce)?"
No PRICE_ACTION levels are promoted. Each bar's low creates a new bucket (if body-position qualifies),
but no bucket receives a second rejection since the price keeps falling. All buckets decay to zero.
This is the correct behavior: no support levels should be detected on a pure sell-off day.

## Open Points

1. **NMS Comparator Bug** (ChatGPT finding): The `.reversed()` chain in `suppressOverdenseZones()` may incorrectly flip the touchCount ordering. This is a pre-existing issue not introduced by ODIN-064. Should be tracked as a separate story.

2. **PRICE_ACTION Level Pruning** (ChatGPT suggestion): Promoted PRICE_ACTION levels with 0 touches persist indefinitely in `activeLevels`. A pruning rule (e.g., remove after N bars with no touches and far from current price) would prevent silent accumulation. Follow-up story recommended.

3. **Pre-Market Bucket Seeding** (Gemini finding): PriceActionTouchMap processes all bars including pre-market. In ODIN's architecture, pre-market bars have volume > 0 (they come from the data feed), so the RTH-start filter in SrLevelEngine doesn't protect PriceActionTouchMap. However, pre-market bars are typically thin and the body-position threshold reduces noise. A stricter mitigation would be to gate PriceActionTouchMap on the same `rthStartBarIndex` as clustering. Low priority.
