# ODIN-029 — Implementierungsprotokoll: Core Daily Performance Recording

**Story:** ODIN-029 — EOD Daily Performance Recording
**Agent:** impl-odin029
**Datum:** 2026-02-22
**Status:** BEREIT FÜR QS

---

## Ergebnis

Alle Akzeptanzkriterien der Story erfüllt. Kompilierung erfolgreich, Unit-Tests (97) und
Integrationstests (7) grün.

---

## Geänderte / neue Dateien

### Neue Dateien

| Datei | Beschreibung |
|-------|-------------|
| `odin-core/src/main/java/de/its/odin/core/service/DailyPerformanceRecorder.java` | Hauptservice EOD-Aggregation |
| `odin-core/src/main/java/de/its/odin/core/service/TradeJournalEntry.java` | Immutable Record für Trade-Journal-Einträge |
| `odin-core/src/test/java/de/its/odin/core/service/DailyPerformanceRecorderTest.java` | 15 Unit-Tests (Surefire) |
| `odin-core/src/test/java/de/its/odin/core/persistence/DailyPerformanceRepositoryIntegrationTest.java` | 7 Integrationstests (Failsafe, Zonky Postgres 14) |
| `odin-core/src/test/resources/application.properties` | Test-Konfiguration für Zonky |
| `odin-core/src/test/resources/db/migration/V000__stub_timescaledb.sql` | No-op TimescaleDB Stub für Flyway |

### Geänderte Dateien

| Datei | Änderung |
|-------|----------|
| `odin-core/pom.xml` | `spring-boot-starter-json` + Zonky Test-Abhängigkeiten hinzugefügt |
| `odin-core/src/main/java/de/its/odin/core/service/GlobalRiskManager.java` | `getRealizedPnl()` und `getTotalCyclesCompleted()` hinzugefügt |
| `odin-core/src/main/java/de/its/odin/core/persistence/DailyPerformanceRepository.java` | `findRecentByModeBeforeDate()` JPQL-Query hinzugefügt |
| `odin-core/src/main/java/de/its/odin/core/persistence/DailyPerformanceEntity.java` | `@UniqueConstraint` Annotation ergänzt (dokumentiert DB-Constraint aus V011) |
| `odin-core/src/main/java/de/its/odin/core/service/LifecycleManager.java` | `DailyPerformanceRecorder` injiziert, `stopPipeline()` + `recordEodPerformance()` extrahiert, `finally`-Block für EOD-Garantie |
| `odin-core/src/main/java/de/its/odin/core/config/CoreConfiguration.java` | `DailyPerformanceRecorder` Bean + aktualisierter `LifecycleManager` Bean |
| `odin-execution/src/main/java/de/its/odin/execution/persistence/TradeRepository.java` | `findByRunIdAndExitTimeIsNotNullOrderByExitTimeAsc()` hinzugefügt |
| `odin-core/src/test/java/de/its/odin/core/service/LifecycleManagerTest.java` | `DailyPerformanceRecorder` Mock ergänzt |
| `odin-core/src/test/java/de/its/odin/core/pipeline/PipelineFactoryTest.java` | Für neue Konstruktor-Signaturen (GateProperties, SizingProperties) angepasst |

### Flyway-Migration

**V027** (`odin-persistence/src/main/resources/db/migration/V027__extend_daily_performance.sql`)
wurde in der Vorsession erstellt und ist bereits committed. Die Migration fügt hinzu:
`win_rate`, `loss_rate`, `avg_win`, `avg_loss`, `profit_factor`, `sharpe_daily`, `cycle_count`, `trade_journal` (JSONB).

---

## Implementierte Metriken

| Metrik | Datenquelle | Berechnung |
|--------|-------------|-----------|
| `totalPnl` | TradeRepository (primär) / GlobalRiskManager (Fallback wenn keine Trades) | Summe `realizedPnl` aller Completed-Trades |
| `totalTrades` | TradeRepository | `COUNT(exitTime IS NOT NULL)` |
| `winningTrades` | TradeRepository | `COUNT(realizedPnl > 0)` |
| `winRate` | Berechnet | `winningTrades / totalTrades` (null wenn keine Trades) |
| `lossRate` | Berechnet | `(totalTrades - winningTrades) / totalTrades` (null wenn keine Trades) |
| `avgWin` | Berechnet | `SUM(positiver PnL) / winCount` (null wenn keine Gewinner) |
| `avgLoss` | Berechnet | `SUM(negativer PnL) / lossCount` (null wenn keine Verlierer) |
| `profitFactor` | Berechnet | `grossWin / |grossLoss|` (null wenn keine Verlust-Trades) |
| `maxDrawdown` | Berechnet | Peak-Highwater-Mark auf exitTime-sortierten Trades |
| `cycleCount` | GlobalRiskManager | `getTotalCyclesCompleted()` |
| `sharpeDaily` | Berechnet + DB-History | Annualisiert, Sample-StdDev (N-1), min. 5 Datenpunkte, 60-Tage Lookback |
| `tradeJournal` | TradeRepository → Jackson | JSON-Array aus `TradeJournalEntry` Records |

---

## Architekturentscheidungen

### Transaktionsmanagement
`TransactionTemplate` statt `@Transactional` — pattern-konform für POJOs in odin-core.
Das `findByTradingDateAndMode` läuft innerhalb der Lambda → Entity bleibt managed.

### totalPnl Fallback-Logik
```
completedTrades.isEmpty() ? totalPnlFromGrm : metrics.totalPnl()
```
Wenn Trades existieren, ist die Trade-Summe autoritativ — auch bei Netto-Null (Break-even Tag).
GRM nur als Fallback wenn überhaupt keine Trades abgeschlossen wurden.

### Sharpe Future-Leakage Protection
`findRecentByModeBeforeDate(mode, tradingDate, pageable)` — JPQL-Filter `tradingDate < :beforeDate`
schließt den aktuellen Tag aus der History aus. Heute's PnL wird nach dem Fetch angehängt.

### Drawdown-Determinismus
Trade-Query mit explizitem `OrderByExitTimeAsc` — Drawdown-Berechnung ist deterministisch unabhängig
von DB-Rückgabereihenfolge.

### BigDecimal.sqrt() für Sharpe
Sharpe-Standardabweichung wird mit `BigDecimal.sqrt(MathContext)` statt `double` berechnet,
um Präzisionsverlust bei Portfolios mit großen PnL-Werten zu vermeiden.

### EOD-Garantie in stopTrading
```java
try {
    for (TradingPipeline pipeline : pipelines) {
        stopPipeline(pipeline);  // per-pipeline try/catch
    }
} finally {
    recordEodPerformance();  // läuft immer
}
```
Fehler in einer Pipeline blockieren weder andere Pipelines noch die EOD-Aufzeichnung.

---

## Tests

### Unit-Tests (Surefire, `*Test`)

| Testklasse | Tests | Ergebnis |
|-----------|-------|---------|
| `DailyPerformanceRecorderTest` | 15 | GRÜN |
| `LifecycleManagerTest` | 7 | GRÜN |
| Alle anderen odin-core Tests | 75 | GRÜN |
| **Gesamt** | **97** | **GRÜN** |

**Coverage (DailyPerformanceRecorderTest):**
- `computeMetrics`: Leer-Trades, gemischte Trades, nur Gewinne, nur Verluste, Profit-Factor-Berechnung
- `computeMaxDrawdown`: Sequenzen mit Gain-first/Loss-first, all-wins, all-losses, leere Liste
- `computeRollingSharpe`: Unzureichende Daten, ausreichende Daten, identische Returns (StdDev=0)
- `buildTradeJournal`: Vollständiges Mapping aller Felder
- `recordDailyPerformance`: Kein vorhandener Record, mit vorhandenem Record (Update), Trades-Szenario

### Integrationstests (Failsafe, `*IntegrationTest`)

| Testklasse | Tests | Ergebnis |
|-----------|-------|---------|
| `DailyPerformanceRepositoryIntegrationTest` | 7 | GRÜN |

**DB:** Zonky embedded Postgres 14.15, Flyway-Migrations V011 + V027 (+ V000 TimescaleDB-Stub)

**Coverage:**
- `findByTradingDateAndMode`: Save & Find, Mode-Filter
- Trade-Journal JSON Round-Trip (JSONB)
- `findByTradingDateBetweenAndModeOrderByTradingDateAsc`: Datumsbereich + Sortierung
- `findRecentByModeBeforeDate`: Limit + Ausschluss aktueller Tag, Mode-Isolation
- Alle V027-Felder vollständig persistiert und geladen

---

## ChatGPT-Review (2 Runden, Owner: impl-odin029)

### Runde 1 — Findings und Umsetzung

| Finding | Severity | Umgesetzt |
|---------|----------|-----------|
| `totalPnl` Fallback-Fehler (net-zero Break-even) | CRITICAL | Ja — `isEmpty()` statt `pnl != 0` |
| Drawdown nicht deterministisch (kein Sort) | CRITICAL | Ja — `OrderByExitTimeAsc` in Query |
| Sharpe Future-Leakage (darf aktuelle Daten einschließen) | IMPORTANT | Ja — `beforeDate` Parameter in JPQL |
| Sample-StdDev (N-1) statt Population-StdDev | IMPORTANT | Ja — `count - 1` im Nenner |
| `stopTrading` nicht robust (Exception bricht EOD ab) | IMPORTANT | Ja — `finally` + `stopPipeline()` |
| Ungenutzter `MathContext` Import | MINOR | Ja — entfernt |

### Runde 2 — Architektur-Klärung

- **Single Run per Day bestätigt:** runId-basierte Trade-Abfrage ist korrekt, kein Multi-Run-Problem
- **JavaDoc "partial run"** entfernt — irreführender Begriff

---

## Gemini-Review (Owner: impl-odin029-gem)

### Dimension 1: Code Bugs

| Finding | Severity | Entscheidung |
|---------|----------|-------------|
| JavaTimeModule fehlt (Instant-Serialisierung) | IMPORTANT | RESOLVED — Spring Boot auto-konfiguriert JavaTimeModule via `spring-boot-starter-json` |
| Precision-Loss bei Sharpe durch `double` | MINOR | Umgesetzt — `BigDecimal.sqrt()` statt `Math.sqrt(double)` |
| JSON-Konkatenation in LifecycleManager | MINOR | Nicht umgesetzt — Integer-Konkatenation ist sicher; ObjectMapper-Abhängigkeit wäre over-engineered |

### Dimension 2: Konzepttreue

| Finding | Severity | Entscheidung |
|---------|----------|-------------|
| Core→Execution als DDD-Verletzung | CRITICAL | RESOLVED — architektonisch korrekt per CLAUDE.md (odin-core ist Orchestrierungsschicht) |
| TransactionTemplate-Einsatz | INFO | Positiv bewertet, keine Aktion nötig |
| UUID als String in RunContext | MINOR | Nicht umgesetzt — out of scope für ODIN-029 |
| `@UniqueConstraint` Annotation fehlt | IMPORTANT | Umgesetzt — `@Table` um `@UniqueConstraint` ergänzt (DB-Constraint aus V011 dokumentiert) |

### Dimension 3: Produktionspraxis

| Finding | Severity | Entscheidung |
|---------|----------|-------------|
| Memory-Risiko bei vielen Trades (alle in RAM) | IMPORTANT | Nicht umgesetzt — per CLAUDE.md "erst bauen, dann messen"; ODIN ist kein HFT-System |
| Thread-Sicherheit LifecycleManager | IMPORTANT | Nicht umgesetzt — sequenzieller Lifecycle in Single-Thread-Kontext; `volatile` ausreichend |
| EOD-Muster (stopPipeline + finally) | INFO | Positiv bewertet |
| Trade-Journal null bei Serialisierungsfehler | MINOR | Nicht umgesetzt — dokumentiertes Verhalten, null ist valider Zustand per Spec |

---

## Vorhandener Bug behoben (Pre-existing Break)

`PipelineFactory.java` kompilierte nicht, weil `DecisionArbiter`-Konstruktor einen
`GateCascadeEvaluator` als ersten Parameter erwartet (durch andere Story geändert).
Fix: `GateCascadeEvaluator` Import + Instanziierung hinzugefügt.

`PipelineFactoryTest.java` ebenfalls für neue Konstruktor-Signaturen angepasst
(`BrainProperties.GateProperties`, `ExecutionProperties.SizingProperties`).

---

## Offene Punkte / Hinweise für QS-Agent

1. **V027 bereits in Git:** Flyway-Migration wurde in der Vorsession erstellt und committed.
   Der QS-Agent muss nur die neuen/geänderten Java-Dateien committen.

2. **`WinRate = 0.000000` bei only-losses:** Das ist korrekt (nicht null) wenn totalTrades > 0.
   Entsprechend getestete Assertions in `DailyPerformanceRecorderTest`.

3. **`profitFactor = null` bei only-wins:** null ist korrekt (kein Nenner). Entsprechend getestet.

4. **JSON-Konkatenation LifecycleManager:** Bewusst nicht geändert (Integer, kein Injektionsrisiko).
   Gemini-Finding MINOR als "Accepted Risk" dokumentiert.

5. **Memory/Trade-in-RAM:** Als bekannte Einschränkung dokumentiert. Für erwartete Trade-Volumina
   (2-3 Instrumente, Long-Only, 5-10 Trades/Tag) kein Problem.
