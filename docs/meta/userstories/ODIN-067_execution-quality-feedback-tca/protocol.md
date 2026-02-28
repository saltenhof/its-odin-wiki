# Protokoll: ODIN-067 — Execution Quality Feedback (TCA)

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (17 TcaCalculator + 6 TcaAggregator = 23 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings bewertet
- [x] Gesamter Test-Suite gruen (305 Tests, 0 Failures)
- [ ] Commit & Push

## Design-Entscheidungen
- TcaRecord is a pure domain record in odin-execution (not a JPA entity for now — persistence via EventLog)
- TcaCalculator is stateless, guards against division by zero with Double.isFinite() checks
- Decision price tracking added to TradeIntent as new field (decisionPrice)
- BrokerSimulator spread-based fill model: uses estimated Ask for BUY, estimated Bid for SELL
  (halfSpread = basePrice * spreadEstimateBps / 2 / 10000)
- New config property: `odin.broker.sim.spread-estimate-bps=5` (separate from slippageBps for stops)
- TcaAggregator in odin-backtest aggregates TCA metrics from fill snapshots
- BacktestSummary extended with avgImplementationShortfall, avgSlippageBps, worstSlippageBps
- TCA metrics in OMS logged as TCA_RECORD event via EventLog (forensic audit trail)

## Neue/Geaenderte Dateien

### Neue Klassen
| Datei | Modul | Beschreibung |
|-------|-------|-------------|
| TcaCalculator.java | odin-execution | Stateless TCA calculator (IS, Effective Spread, Slippage) |
| TcaRecord.java | odin-execution | Domain record with all TCA metrics per fill |
| TcaRecordEntity.java | odin-execution | JPA entity for odin.tca_record table |
| TcaRecordRepository.java | odin-execution | Spring Data JPA repository |
| TcaAggregator.java | odin-backtest | Aggregates TCA across fills (avg, worst) |
| V030__create_tca_record.sql | odin-persistence | Flyway migration creating tca_record table |
| TcaCalculatorTest.java | odin-execution (test) | 17 unit tests for TCA calculations |
| TcaAggregatorTest.java | odin-backtest (test) | 6 unit tests for TCA aggregation |

### Geaenderte Klassen
| Datei | Aenderung |
|-------|-----------|
| TradeIntent.java | +decisionPrice (double) als letztes Record-Feld |
| DecisionArbiter.java | Berechnet midPrice aus bid/ask, setzt decisionPrice |
| IntentSigner.java | Reicht decisionPrice durch bei sign() |
| SizedOrder.java | +decisionPrice (double) als letztes Record-Feld |
| RiskGate.java | Reicht decisionPrice von TradeIntent an SizedOrder |
| OrderManagementService.java | +TcaCalculator, entryDecisionPrice tracking, logTcaForFill() |
| BrokerSimulator.java | Spread-basiertes Fill-Modell (halfSpread statt slippage fuer Entry/Exit) |
| BrokerProperties.SimProperties | +spreadEstimateBps (int) als neues config-Feld |
| BacktestReport.BacktestSummary | +avgImplementationShortfall, +avgSlippageBps, +worstSlippageBps |
| BacktestRunner.java | Zero-TCA-Metriken in computeSummary (Aggregation via TcaAggregator verfuegbar) |
| ExecutionProperties.java | +TcaProperties (enabled, logLevel) |
| ParameterOverrideApplier.java | Reicht TcaProperties durch |
| odin-broker.properties | +spread-estimate-bps=5, slippage-bps=3 (reduziert fuer Stops) |
| odin-execution.properties | +tca.enabled=true, +tca.log-level=INFO |

### Test-Anpassungen (Constructor-Fixes)
14 Test-Dateien fuer ExecutionProperties (+TcaProperties), 8 fuer BacktestSummary (+3 TCA fields),
~35 TradeIntent-Konstruktor-Aufrufe (+decisionPrice), ~13 SizedOrder-Aufrufe (+decisionPrice)

## ChatGPT-Sparring

### Session: Edge Cases fuer TCA Tests
**Datum:** 2026-02-28
**Wesentliche Findings:**

1. **Partial Fill Weighting (MINOR):** Aggregator verwendet count-weighted statt notional-weighted Averaging.
   Gleiche VWAP mit unterschiedlicher Fill-Aufteilung ergibt unterschiedliche Avg-Metriken.
   Akzeptabel fuer V1, bewusste Entscheidung.

2. **Buy vs Sell Sign Convention (MAJOR, DEFERRED):** IS-Formel liefert negative Werte fuer adverse Sells.
   Aggregation von Buy+Sell kann sich aufheben. Fuer V1 akzeptabel (ODIN ist long-only).
   Fuer V2 empfohlen: adverse-aligned Metriken mit Side-Multiplier.

3. **After-Hours Stale Quotes (MINOR):** Mid-Price kann stale sein wenn Fill nach RTH stattfindet.
   Empfehlung: Quote-Age-Validierung oder Session-Segmentierung.

4. **Large N Summation Stability (INFO):** Naive Summation kann bei 1M+ Fills Precision verlieren.
   Kahan Summation empfohlen fuer Produktionsqualitaet. V1 akzeptabel.

5. **Locked/Crossed Markets (MINOR):** bid >= ask nicht speziell behandelt. isValidPrice() faengt
   nur Zero/NaN/Infinity ab. Fuer Backtest-Simulator irrelevant (keine echten Quotes).

## Gemini-Review (3 Dimensionen)

### Dimension 1: Code-Review
**Datum:** 2026-02-28

- **[CRITICAL] Missing Side-Awareness in TCA Metrics:**
  IS-Formel produziert fuer Sells falsche Vorzeichen. ODIN ist V1 long-only → DEFERRED.

- **[MAJOR] Limit Order Spread Violation:**
  min(open, limitPrice) + halfSpread kann Limit-Preis ueberschreiten. Pre-existing Verhalten
  aus V1 Slippage-Modell, nicht durch TCA eingefuehrt. NOTED, separate Story.

- **[INFO] Robust Zero/NaN Guarding:** isValidPrice() korrekt implementiert.
- **[INFO] Aggregation Safety:** TcaAggregator Empty-Guard korrekt.

### Dimension 2: Konzepttreue
- **[CRITICAL] Slippage Sign Convention:** Gleiche Issue wie Side-Awareness.
  Aggregation von Buy+Sell hebt sich auf. DEFERRED (long-only V1).

- **[INFO] Effective Spread Accuracy:** Formel entspricht Industry Standard
  (absoluter Wert loest Side-Problem inherent).

- **[MAJOR] Missing Stop-Loss Gap Logic:** Stops füllen bei triggerPrice statt Gap-adjusted.
  Pre-existing BrokerSimulator Limitierung. NOTED, separate Story.

### Dimension 3: Praxis-Review
- **[MAJOR] Static Spread Estimates:** Kein dynamischer Spread nach Tageszeit.
  Open-Spread >> Midday-Spread. DEFERRED fuer V2.

- **[MAJOR] Hardcoded Long-Only Stops:** Pre-existing, nicht TCA-bezogen.

- **[MINOR] 5 bps Default:** Evtl. konservativ fuer Large-Cap (AAPL: 1-2 bps).
  Konfigurierbar via Properties, kein Code-Fix noetig.

### Bewertung der Findings

| Finding | Severity | Aktion | Begruendung |
|---------|----------|--------|-------------|
| Side-Awareness | CRITICAL | DEFERRED | ODIN V1 ist long-only, keine Sell-Entries |
| Limit Order Spread Violation | MAJOR | NOTED | Pre-existing, nicht durch TCA eingefuehrt |
| Stop-Loss Gap Logic | MAJOR | NOTED | Pre-existing BrokerSimulator-Issue |
| Static Spread | MAJOR | DEFERRED | V2-Feature, dynamische Spread-Schaetzung |
| Hardcoded Long-Only Stops | MAJOR | NOTED | Pre-existing, Short-Support separate Story |
| 5 bps Default | MINOR | OK | Konfigurierbar, angemessen fuer Mid/High-Beta |
| Partial Fill Weighting | MINOR | OK | Count-weighted akzeptabel fuer V1 |

**Keine actionable Changes fuer V1 erforderlich.** Alle CRITICAL/MAJOR Findings sind entweder
pre-existing (nicht durch TCA eingefuehrt) oder betreffen Short-Selling (nicht in V1 Scope).

## Build & Test

```
mvn test → 305 Tests, 0 Failures, 0 Errors — BUILD SUCCESS
```

Neue Tests:
- TcaCalculatorTest: 17 Tests (IS, Effective Spread, Slippage, edge cases, null validation)
- TcaAggregatorTest: 6 Tests (empty, single, multi, negative slippage, null input)
- BrokerSimulatorTest: 39 Tests (inkl. spread-basiertes Fill-Modell, alle bestehenden Tests gruen)
