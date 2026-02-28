# ODIN-067: Execution Quality Feedback (TCA)

## Modul

odin-execution, odin-api, odin-broker, odin-backtest

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

L

## Kontext

Transaction Cost Analysis (TCA) misst die Differenz zwischen dem theoretischen Entscheidungspreis und dem tatsaechlichen Fill-Preis — inklusive Slippage, Spread und Market Impact. Ohne dieses Feedback weiss ODIN nicht, ob Signale nach realen Kosten profitabel sind. Das ist die groesste einzelne Quelle fuer Diskrepanz zwischen Simulation und Live. Diese Story fuehrt Implementation Shortfall als Leitmetrik ein, persistiert TCA-Daten pro Order und verbessert das Backtest-Fill-Modell auf Spread-basiert.

## Scope

### In Scope

- **Implementation Shortfall pro Order:** `IS = (fillPrice - decisionPrice) / decisionPrice` als primaere TCA-Metrik
- **Effective Spread:** `effectiveSpread = 2 * |fillPrice - midPrice| / midPrice` pro Fill
- **Decision-Price Tracking:** Der Preis zum Zeitpunkt der Arbiter-Entscheidung (nicht der Order-Submission) wird als Referenz persistiert
- **TCA-Entity fuer Persistierung:** Pro Trade/Order die TCA-Metriken speichern (Flyway-Migration)
- **Realistisches Backtest-Fill-Modell:** Spread-basiert statt Mid-Price (BrokerSimulator-Erweiterung)
- **Aggregierte TCA-Metriken im BacktestReport:** Avg Slippage, Avg Implementation Shortfall, Worst-Case Slippage

### Out of Scope

- TCA-Feedback als Input in SizeModifier oder Liquidity-Gate (spaetere Story, Thema 19)
- Market-Impact-Modell (Almgren-Chriss) — zu komplex fuer V1
- TCA-Dashboard im Frontend (separate Story)
- Adaptive Order-Typ-Selektion basierend auf TCA-Ergebnissen (Thema 19)

## Akzeptanzkriterien

- [ ] `TradeIntent` (`de.its.odin.api.dto.TradeIntent`) enthaelt neues Feld `decisionPrice` (double): der Mid-Price zum Zeitpunkt der Arbiter-Entscheidung
- [ ] `TcaRecord` als neues Record in odin-execution: `instrumentId`, `orderId`, `decisionPrice`, `fillPrice`, `midPriceAtFill`, `implementationShortfall`, `effectiveSpread`, `slippageBps`, `fillTimestamp`
- [ ] `TcaCalculator` berechnet: `implementationShortfall = (fillPrice - decisionPrice) / decisionPrice` (positiv = unguenstig fuer Long-Entry)
- [ ] `TcaCalculator` berechnet: `effectiveSpread = 2 * abs(fillPrice - midPriceAtFill) / midPriceAtFill`
- [ ] `TcaCalculator` berechnet: `slippageBps = (fillPrice - midPriceAtFill) / midPriceAtFill * 10000` (Basis Points)
- [ ] `OrderManagementService` ruft `TcaCalculator` nach jedem Fill auf und loggt `TcaRecord` via EventLog
- [ ] `BrokerSimulator` (`de.its.odin.broker.sim.BrokerSimulator`) verwendet Spread-basiertes Fill-Modell: `fillPrice = askPrice` fuer Buys (statt Mid-Price)
- [ ] `BacktestReport.BacktestSummary` enthaelt neue Felder: `avgImplementationShortfall`, `avgSlippageBps`, `worstSlippageBps`
- [ ] Unit-Test verifiziert: IS-Berechnung fuer decisionPrice=100.00, fillPrice=100.05 ergibt IS=0.0005 (5 bps)
- [ ] Unit-Test verifiziert: effectiveSpread-Berechnung fuer fillPrice=100.05, midPrice=100.00 ergibt 0.001 (10 bps)
- [ ] Flyway-Migration erstellt `tca_record`-Tabelle

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `TcaCalculator` | odin-execution | `de.its.odin.execution.tca` | Stateless Calculator. Input: decisionPrice, fillPrice, midPriceAtFill. Output: TcaRecord |
| `TcaRecord` | odin-execution | `de.its.odin.execution.tca` | Record mit allen TCA-Metriken pro Fill |
| `TcaRecordEntity` | odin-execution | `de.its.odin.execution.persistence` | JPA-Entity fuer Persistierung (Flyway-Migration erforderlich) |
| `TcaRecordRepository` | odin-execution | `de.its.odin.execution.persistence` | Spring Data JPA Repository |
| `TcaAggregator` | odin-backtest | `de.its.odin.backtest.tca` | Aggregiert TCA-Metriken ueber alle Trades eines Backtest-Runs |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `TradeIntent` (`de.its.odin.api.dto.TradeIntent`) | Neues Feld `decisionPrice` (double) |
| `DecisionArbiter` (`de.its.odin.brain.arbiter.DecisionArbiter`) | Setzt `decisionPrice` in TradeIntent auf aktuellen Mid-Price bei Entscheidung |
| `OrderManagementService` (`de.its.odin.execution.oms.OrderManagementService`) | Nach Fill: TcaCalculator aufrufen, TcaRecord loggen |
| `BrokerSimulator` (`de.its.odin.broker.sim.BrokerSimulator`) | Fill-Preis = Ask fuer Buys, Bid fuer Sells (statt Mid-Price) |
| `BacktestReport.BacktestSummary` (`de.its.odin.backtest.BacktestReport`) | Neue Felder: `avgImplementationShortfall`, `avgSlippageBps`, `worstSlippageBps` |
| `BacktestRunner` (`de.its.odin.backtest.BacktestRunner`) | TCA-Aggregation nach Backtest-Lauf |

### Flyway-Migration

```sql
CREATE TABLE tca_record (
    id              BIGSERIAL PRIMARY KEY,
    run_id          VARCHAR(64)    NOT NULL,
    instrument_id   VARCHAR(32)    NOT NULL,
    order_id        VARCHAR(64)    NOT NULL,
    decision_price  DECIMAL(12,4)  NOT NULL,
    fill_price      DECIMAL(12,4)  NOT NULL,
    mid_price_at_fill DECIMAL(12,4) NOT NULL,
    implementation_shortfall DECIMAL(10,6) NOT NULL,
    effective_spread DECIMAL(10,6) NOT NULL,
    slippage_bps    DECIMAL(8,2)   NOT NULL,
    fill_timestamp  TIMESTAMPTZ    NOT NULL,
    created_at      TIMESTAMPTZ    DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_tca_record_run ON tca_record(run_id);
CREATE INDEX idx_tca_record_instrument ON tca_record(instrument_id);
```

### Konfiguration

```properties
odin.execution.tca.enabled=true
odin.execution.tca.log-level=INFO
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 3: Execution-Quality-Feedback (TCA / Implementation Shortfall), Abschnitte IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/07-oms.md` — OMS-Architektur, Fill-Handling
- `docs/backend/architecture/03-broker-integration.md` — BrokerSimulator, Fill-Modell

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, DDD-Modulschnitt (Entity lebt in odin-execution)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, Transaktionsgrenzen (TransactionTemplate statt @Transactional fuer Pro-Pipeline-POJOs)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution,odin-api,odin-broker,odin-backtest`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (TcaRecord)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.execution.tca.*`

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `TcaCalculator` mit bekannten Werten
- [ ] Unit-Tests fuer `TcaAggregator`
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer EventLog, BrokerGateway

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: OrderManagementService mit echtem TcaCalculator und Mock-EventLog
- [ ] Integrationstest: BrokerSimulator Spread-basiertes Fill-Modell validiert
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank
- [ ] Embedded-Postgres-Test mit Zonky fuer TcaRecordEntity/Repository
- [ ] Flyway-Migration wird im Test ausgefuehrt

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen und Akzeptanzkriterien
- [ ] Edge Cases diskutiert (z.B. Partial Fills, After-Hours-Fills, Spread = 0)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Numerik, Division-by-Zero, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (Implementation Shortfall Definition)
- [ ] Dimension 3: Praxis-Review (realistische Spread-Annahmen, Market-Hours)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Decision-Price ist der Mid-Price zum Zeitpunkt der Arbiter-Entscheidung**, nicht der Order-Submission. Die Differenz dazwischen ist typischerweise < 100ms, aber in schnellen Maerkten relevant.
- **BrokerSimulator Fill-Modell:** Aktuell wird vermutlich Mid-Price als Fill-Price verwendet. Die Aenderung auf Ask (fuer Buys) macht Backtests realistischer aber auch pessimistischer. Das ist gewuenscht.
- **Spread-Daten im Backtest:** Pruefen ob `MarketSnapshot` im Backtest-Modus Bid/Ask enthaelt. Wenn nur OHLCV vorhanden, kann der Spread aus historischen Durchschnittswerten geschaetzt werden (z.B. 0.05% fuer Large-Cap US-Equities).
- **TcaRecord-Persistierung:** Im Live-Modus via Repository persistieren. Im Backtest-Modus optional (konfigurierbar), da es die Backtest-Laufzeit erhoehen kann.
- **Division by Zero:** `decisionPrice` und `midPriceAtFill` koennen theoretisch 0 sein (Pre-Market, fehlerhafte Daten). Guard mit Double.isFinite()-Checks.
