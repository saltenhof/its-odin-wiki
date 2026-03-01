# ODIN-084: Market Data Level-Aware Gates

**Epic:** Backtest-Feedback Optimization

## Modul

odin-api, odin-data, odin-brain, odin-core

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

M

---

## Kontext

During the IREN backtest (2026-02-23) the VOLUME gate blocked entry at 9:40 EST despite the stock having 2.6M volume on the second RTH bar. Root cause: `volumeRatio = null/NaN` because pre-market bars carry no volume in historical data, making the ratio calculation unable to establish a reference. The entry was delayed by 5 minutes (entry at 40.64 instead of potentially 40.34 = ~390 USD lost alpha on 1301 shares).

The fundamental problem: gates behave identically regardless of the data quality available. But data quality varies by mode and data source. Historical data feeds typically provide OHLCV bars without extended-hours volume (pre/post-market volume = 0). Live feeds may include Level-1 quotes (bid/ask/spread) or even Level-2 depth. Gates that require data not available at the configured level should gracefully degrade (skip), while gates that have sufficient data should enforce strictly.

This story introduces a `MarketDataLevel` concept that makes the gate cascade data-quality-aware: gates know which data level they require, and the system knows which level is available. Gates requiring unavailable data are skipped; gates with available data enforce strictly (NaN = real error).

## Scope

### In Scope

- New enum `MarketDataLevel` in `de.its.odin.api.model` with four levels: `L0_NO_EXTENDED_VOLUME`, `L0_FULL_VOLUME`, `L1_QUOTES`, `L2_DEPTH`
- New configuration property `odin.data.market-data-level` (default: `L0_NO_EXTENDED_VOLUME` for backtests, `L1_QUOTES` for live)
- Each gate declares two level methods: `minimumLevelToEvaluate()` (hard skip threshold) and `minimumLevelForStrictEnforcement()` (strict NaN handling threshold)
- `GateCascadeEvaluator` receives the configured `MarketDataLevel` and uses the two-method protocol to determine gate behavior: hard skip, graceful degradation, or strict enforcement
- `VolumeGate` behavior at Level 0 (no extended volume): when `volumeRatio = NaN` (i.e., `Double.isNaN(volumeRatio)`), the gate returns PASS with a warning instead of blocking. Once volumeRatio is computable (non-NaN), the gate enforces normally using standard gate logic
- `VolumeGate` behavior at Level 1+ (full volume): NaN is treated as a real error and the gate blocks (FAIL)
- `SpreadGate` declares both minimum levels as `L1_QUOTES` — automatically skipped when running on L0 data
- `OfiGate` (if present) declares both minimum levels as `L2_DEPTH` — automatically skipped when running below L2
- `GateResult` includes a new status `SKIPPED` with a reason string; SKIPPED gates are treated as PASS for cascade decision purposes (they do not block entry)
- Configuration wiring in `DataProperties` (odin-data) or `BrainProperties` (odin-brain)

### Out of Scope

- Automatic detection of the data level (it is manually configured per mode/profile)
- Implementation of new L2 gates (OFI gate already exists separately in ODIN-069)
- Changes to the LLM prompt (handled separately in Iteration 6)
- Frontend display of the active data level
- Dynamic data-level switching during a running session

## Akzeptanzkriterien

- [ ] New enum `MarketDataLevel` in `de.its.odin.api.model` with values: `L0_NO_EXTENDED_VOLUME`, `L0_FULL_VOLUME`, `L1_QUOTES`, `L2_DEPTH`
- [ ] Enum values are ordered (each level implies all capabilities of lower levels) with `includes(MarketDataLevel other)` method
- [ ] New configuration property `odin.data.market-data-level` with default `L0_NO_EXTENDED_VOLUME`
- [ ] Gate interface (e.g. `EntryGate`) extended with two default methods: `minimumLevelToEvaluate()` returning `MarketDataLevel.L0_NO_EXTENDED_VOLUME` and `minimumLevelForStrictEnforcement()` returning `MarketDataLevel.L0_NO_EXTENDED_VOLUME`
- [ ] `VolumeGate` overrides `minimumLevelToEvaluate()` to return `L0_NO_EXTENDED_VOLUME` (always evaluated) and `minimumLevelForStrictEnforcement()` to return `L0_FULL_VOLUME` (strict NaN handling only at L0_FULL_VOLUME or above)
- [ ] `VolumeGate` at Level 0: when `Double.isNaN(volumeRatio)` is true, returns `GateResult.PASS` with warning message "Volume ratio unavailable (no extended-hours data at L0), graceful degradation"
- [ ] `VolumeGate` at Level 0: when `volumeRatio` is computable (non-NaN), enforces the volume ratio gate normally using standard gate logic (strict enforcement)
- [ ] `VolumeGate` at Level 1+: NaN `volumeRatio` is treated as a genuine data error and returns `GateResult.FAIL`
- [ ] `SpreadGate` overrides both `minimumLevelToEvaluate()` and `minimumLevelForStrictEnforcement()` to return `L1_QUOTES`
- [ ] `OfiGate` (if exists) overrides both methods to return `L2_DEPTH`
- [ ] `GateCascadeEvaluator` checks `configuredLevel.includes(gate.minimumLevelToEvaluate())` before evaluating each gate; if not: gate is hard-skipped with status `SKIPPED`, logged, and included in the cascade result
- [ ] `GateCascadeEvaluator` passes the configured level and the gate's `minimumLevelForStrictEnforcement()` to each gate so it can determine whether to apply graceful degradation or strict NaN handling
- [ ] `GateResult` has a new status `SKIPPED` (in addition to `PASS` and `FAIL`) with a reason string
- [ ] A `SKIPPED` gate is treated as `PASS` for cascade decision purposes — it does not block entry
- [ ] `GateCascadeResult` includes the count and list of skipped gates
- [ ] Configuration is mode-aware: backtest default = `L0_NO_EXTENDED_VOLUME`, live default = `L1_QUOTES` (via Spring profiles or property override)
- [ ] Unit-Test: `MarketDataLevel.L1_QUOTES.includes(L0_FULL_VOLUME)` returns `true`
- [ ] Unit-Test: `MarketDataLevel.L0_NO_EXTENDED_VOLUME.includes(L1_QUOTES)` returns `false`
- [ ] Unit-Test: VolumeGate at L0 with NaN volumeRatio returns PASS with warning
- [ ] Unit-Test: VolumeGate at L0 with valid (non-NaN) volumeRatio enforces normal gate logic
- [ ] Unit-Test: VolumeGate at L1 with NaN volumeRatio returns FAIL
- [ ] Unit-Test: GateCascadeEvaluator at L0 skips SpreadGate (requires L1) and evaluates VolumeGate
- [ ] Unit-Test: GateCascadeEvaluator at L1 evaluates both SpreadGate and VolumeGate
- [ ] Unit-Test: GateCascadeEvaluator treats SKIPPED gates as PASS when determining cascade result
- [ ] Regression-Test (IREN scenario): Given Level=L0_NO_EXTENDED_VOLUME and a backtest with pre-market bars having volume=0, when the first RTH bars produce NaN volumeRatio, then VolumeGate returns PASS (graceful degradation) instead of FAIL
- [ ] Integrationstest: Full gate cascade evaluation with configured MarketDataLevel, verifying correct skip/evaluate behavior across all gates

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `MarketDataLevel` | odin-api | `de.its.odin.api.model` | Enum with 4 levels. Ordered. Method `includes(MarketDataLevel other)` returns true if `this.ordinal() >= other.ordinal()`. Methods `hasExtendedVolume()`, `hasQuotes()`, `hasDepth()` for convenience |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `EntryGate` (`de.its.odin.api.port` or `de.its.odin.brain.quant.gates`) | Two new default methods: `minimumLevelToEvaluate()` returning `L0_NO_EXTENDED_VOLUME` (gate is always evaluated unless overridden) and `minimumLevelForStrictEnforcement()` returning `L0_NO_EXTENDED_VOLUME` (gate is always strict unless overridden) |
| `VolumeGate` (`de.its.odin.brain.quant.gates.VolumeGate`) | Override `minimumLevelToEvaluate()` to `L0_NO_EXTENDED_VOLUME` (always evaluated). Override `minimumLevelForStrictEnforcement()` to `L0_FULL_VOLUME`. Gate logic: when configured level is below strict enforcement level and `Double.isNaN(volumeRatio)`, return PASS with warning. When volumeRatio is non-NaN, apply normal gate logic regardless of level. When at/above strict enforcement level and NaN, return FAIL |
| `SpreadGate` (`de.its.odin.brain.quant.gates.SpreadGate`) | Override both `minimumLevelToEvaluate()` and `minimumLevelForStrictEnforcement()` to `L1_QUOTES` |
| `OfiGate` (`de.its.odin.brain.quant.gates.OfiGate`) | Override both methods to `L2_DEPTH` (if this gate exists from ODIN-069) |
| `GateCascadeEvaluator` (`de.its.odin.brain.quant.GateCascadeEvaluator`) | Constructor receives `MarketDataLevel`. Before evaluating each gate, check `configuredLevel.includes(gate.minimumLevelToEvaluate())`. If not: add SKIPPED result, log, continue. If yes: evaluate gate, passing level context so the gate can determine strict vs. graceful behavior. When computing the cascade result, treat SKIPPED as PASS (does not block) |
| `GateResult` (`de.its.odin.brain.quant.gates` or `de.its.odin.api.dto`) | Add `SKIPPED` status alongside existing `PASS`/`FAIL`. Include `reason` string for SKIPPED results |
| `GateCascadeResult` (`de.its.odin.brain.quant`) | New field `skippedGates` (list), `skippedCount` (int). Cascade decision: SKIPPED gates are treated as PASS and do not block entry |
| `DataProperties` (`de.its.odin.data.config.DataProperties`) | New field `marketDataLevel` (`MarketDataLevel`, default `L0_NO_EXTENDED_VOLUME`) |
| `PipelineFactory` (`de.its.odin.core.pipeline.PipelineFactory`) | Pass `MarketDataLevel` from configuration to `GateCascadeEvaluator` and individual gates |

### Two-Method Gate Protocol

Each gate declares two level thresholds that together define the gate's behavior across the data level spectrum:

1. **`minimumLevelToEvaluate()`** — The minimum `MarketDataLevel` at which the gate is evaluated at all. Below this level, the gate is **hard-skipped** (status = `SKIPPED`). The data required by this gate is fundamentally absent.

2. **`minimumLevelForStrictEnforcement()`** — The minimum `MarketDataLevel` at which the gate treats NaN/null as a genuine error (FAIL). Below this level but at/above `minimumLevelToEvaluate()`, the gate runs in **graceful degradation** mode: NaN values that are known artifacts of the data level limitation produce PASS with warning instead of FAIL. Non-NaN values are always enforced strictly.

**Invariant:** `minimumLevelForStrictEnforcement() >= minimumLevelToEvaluate()` — a gate cannot enforce strictly at a level where it is not even evaluated.

**Decision flow per gate:**

```
configuredLevel.includes(gate.minimumLevelToEvaluate())?
├── NO  → SKIPPED (hard skip, treated as PASS for cascade)
└── YES → Evaluate gate
           configuredLevel.includes(gate.minimumLevelForStrictEnforcement())?
           ├── YES → Strict mode (NaN = FAIL)
           └── NO  → Graceful degradation (NaN = PASS with warning, non-NaN = strict)
```

### Gate-Level Matrix

| Gate | minimumLevelToEvaluate | minimumLevelForStrictEnforcement | Behavior below strict level | Behavior at/above strict level |
|------|------------------------|--------------------------------|---------------------------|-------------------------------|
| `PriceGate` | L0_NO_EXTENDED_VOLUME | L0_NO_EXTENDED_VOLUME | N/A (always strict) | Strict |
| `AtrGate` | L0_NO_EXTENDED_VOLUME | L0_NO_EXTENDED_VOLUME | N/A (always strict) | Strict |
| `VolumeGate` | L0_NO_EXTENDED_VOLUME | L0_FULL_VOLUME | NaN volumeRatio → PASS with warning; valid (non-NaN) ratio → strict enforcement (normal gate logic) | NaN → FAIL (strict) |
| `SpreadGate` | L1_QUOTES | L1_QUOTES | SKIPPED (hard skip at L0) | Strict |
| `OfiGate` | L2_DEPTH | L2_DEPTH | SKIPPED (hard skip below L2) | Strict |

### SKIPPED = PASS for Cascade Decision

A gate with status `SKIPPED` is treated as `PASS` when the `GateCascadeEvaluator` computes the overall cascade result. A skipped gate does not block entry. This is the correct semantic: if the gate's required data is not available, the gate has no opinion and should not prevent trading. The cascade result only blocks entry if at least one **evaluated** gate returns `FAIL`.

### NaN Detection

The primary condition for graceful degradation in VolumeGate is `!Double.isNaN(volumeRatio)`:

- If `volumeRatio` is **non-NaN** (computable): the gate enforces normally using standard gate logic, regardless of the configured data level.
- If `volumeRatio` is **NaN** and the configured level is **below** `minimumLevelForStrictEnforcement()`: the gate returns PASS with a warning (graceful degradation). The NaN is a known artifact of the L0 data limitation (no pre-market volume reference).
- If `volumeRatio` is **NaN** and the configured level is **at or above** `minimumLevelForStrictEnforcement()`: the gate returns FAIL (strict enforcement). The NaN indicates a genuine data pipeline issue.

No bar-counter is needed. The mathematics of the ratio calculation naturally determines when the ratio becomes computable. If a configurable minimum RTH bar count is desired as an additional safety net, it can be added later but is not required by this story.

### Konfiguration

```properties
# Market data level: L0_NO_EXTENDED_VOLUME, L0_FULL_VOLUME, L1_QUOTES, L2_DEPTH
odin.data.market-data-level=L0_NO_EXTENDED_VOLUME
```

Profile-specific overrides:
```properties
# application-sim.properties
odin.data.market-data-level=L0_NO_EXTENDED_VOLUME

# application-live.properties
odin.data.market-data-level=L1_QUOTES
```

### Design-Entscheidung: Graceful Degradation vs. Hard Skip

Two distinct behaviors depending on the relationship between gate and level, governed by the two-method protocol:

1. **Hard Skip (SKIPPED):** The configured level is below `minimumLevelToEvaluate()`. The gate requires data that is fundamentally absent at the configured level (e.g., SpreadGate at L0 — no bid/ask data exists). The gate is not evaluated at all. Treated as PASS for cascade decision.
2. **Graceful Degradation (PASS with warning):** The configured level is at or above `minimumLevelToEvaluate()` but below `minimumLevelForStrictEnforcement()`. The gate runs but tolerates NaN/null for specific computations as a known artifact of the data level limitation (e.g., VolumeGate at L0 with NaN volumeRatio because pre-market has no volume). Non-NaN values are still enforced strictly.
3. **Strict Enforcement:** The configured level is at or above `minimumLevelForStrictEnforcement()`. The gate runs with full strictness. NaN is treated as a real error (FAIL).

This distinction is important: VolumeGate is NOT skipped at L0 — it still evaluates volume during RTH. It only tolerates NaN volumeRatio when it stems from the known L0 limitation (missing pre-market volume).

### Backward Compatibility

Default configuration (`L0_NO_EXTENDED_VOLUME`) will change existing backtest behavior: NaN volumeRatio no longer blocks entry. This is the intended fix — the IREN scenario where a valid entry was blocked due to NaN caused by missing pre-market volume data is exactly the problem this story solves. Gate thresholds and logic for non-NaN values remain unchanged.

## Konzept-Referenzen

- `docs/backend/architecture/04-kpi-engine.md` — Section 2: "Stellung im Decision-Cycle" (gate position in pipeline), Section 5: Quant Validation (gate integration with scoring model)
- `docs/backend/architecture/02-realtime-pipeline.md` — Section 5: "Data Quality Gates" (DQ gate chain, gate results: PASS/WARN/REJECT/ESCALATE), Section 7: "MarketSnapshot" (snapshot contents, DataFlag for approximated values)
- `docs/backend/architecture/06-rules-engine.md` — Section 3: "Regime-Confidence-Gate" (gate as entry precondition), Section 7: "Decision Arbiter" (gate cascade → TradeIntent → Risk Gate flow)
- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 6 (OFI), Thema 1 (Seasonality) for context on data-dependent gates

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package conventions, dependency rules (api has no dependencies, brain depends only on api)
- `docs/backend/guardrails/development-process.md` — Phases, checkpoints, review obligations
- `CLAUDE.md` — Coding rules (ENUM for finite sets, no var, Records for DTOs, JavaDoc on all public members)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api,odin-data,odin-brain,odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (`MarketDataLevel`)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.data.market-data-level` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `MarketDataLevel` (ordering, includes-logic, convenience methods)
- [ ] Unit-Tests fuer `VolumeGate` L0 vs. L1 behavior (NaN handling, non-NaN enforcement)
- [ ] Unit-Tests fuer `GateCascadeEvaluator` skip logic and SKIPPED-as-PASS cascade semantics
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Full gate cascade with real gate implementations and configured MarketDataLevel
- [ ] Integrationstest: Pipeline evaluation at L0 correctly skips L1/L2 gates and tolerates L0 NaN
- [ ] Regression-Test: IREN scenario — L0 with pre-market volume=0, NaN volumeRatio on first RTH bars produces PASS (not FAIL)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Gate-Evaluierung mit MarketDataLevel End-to-End durchspielt

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: Klasse(n), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch ChatGPT — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, NaN-Handling, Enum-Ordnung, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (Implementierung vs. Gate-Level-Matrix, Two-Method Protocol)
- [ ] Dimension 3: Praxis-Review (Edge Cases bei Level-Wechsel, Backtest vs. Live Diskrepanz)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten (Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, ChatGPT-Review)

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Two-method gate protocol is the key design.** Each gate declares `minimumLevelToEvaluate()` (hard skip boundary) and `minimumLevelForStrictEnforcement()` (strict NaN handling boundary). The invariant `minimumLevelForStrictEnforcement() >= minimumLevelToEvaluate()` must hold. For most gates both methods return the same value (e.g., SpreadGate returns L1_QUOTES for both). VolumeGate is the special case where they differ.
- **VolumeGate dual behavior is the key complexity.** VolumeGate is NOT simply skipped at L0. It runs normally during RTH once volumeRatio is computable (non-NaN). The L0-specific tolerance only applies when `Double.isNaN(volumeRatio)` — when the ratio is NaN because there is no pre-market volume to reference. Once the ratio is non-NaN, standard gate logic applies regardless of level. The implementation must distinguish between "NaN because L0 has no pre-market volume" (tolerate) and "NaN because of a data pipeline bug" (fail at L1+).
- **NaN detection is simple: `Double.isNaN(volumeRatio)`.** No bar-counter is needed. The ratio calculation naturally transitions from NaN to a computable value as RTH bars accumulate.
- **SKIPPED is not PASS in status, but equivalent for cascade logic.** A skipped gate must be clearly distinguishable from a passing gate in logs and in the cascade result (`GateResult` status is `SKIPPED`, not `PASS`). However, for the cascade decision (does any gate block entry?), SKIPPED is treated as non-blocking — same effect as PASS.
- **Ordering of `MarketDataLevel` matters.** The enum ordinal defines capability ordering. `L0_NO_EXTENDED_VOLUME < L0_FULL_VOLUME < L1_QUOTES < L2_DEPTH`. The `includes()` method relies on this ordering.
- **Configuration per Spring profile.** The property `odin.data.market-data-level` should have sensible defaults per profile (sim = L0, live = L1). The backtest runner may override this based on the data source being used.
- **Backward compatibility change is intentional.** Default L0 with graceful NaN handling means backtests that previously blocked on NaN volumeRatio will now pass. This is the desired fix for the IREN scenario. Gate thresholds and logic for non-NaN values remain unchanged.
- **MarketClock:** This story does not involve time-based decisions beyond what the existing gates already do. No new `Instant.now()` calls needed.
- **Do not change existing gate thresholds.** This story only adds data-level awareness. Gate thresholds (volume ratio limits, spread limits, OFI z-score limits) remain unchanged.
- **DoD 2.6 uses ChatGPT (not Gemini).** The user-story-specification.md references Gemini in section 2.6, but per project decision (MEMORY.md) Gemini-Review is replaced by ChatGPT-Review. The DoD in this story correctly uses ChatGPT.
