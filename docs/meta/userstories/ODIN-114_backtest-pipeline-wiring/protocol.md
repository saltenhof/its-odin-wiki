# Protocol — ODIN-114: Backtest-Pipeline-Wiring

## Story
ODIN-114 — BacktestEventPublisher mit BacktestRunner und SimulationRunner verdrahten, sodass Backtest-Events an den SSE-Stream fließen.

## Status
**CONDITIONAL PASS** (QA-Report R1: 2026-03-05)

---

## Implementierungs-Chronologie

### Commit `cd3ec52` (2026-03-05)
**feat(backtest): ODIN-114 wire BacktestEventPublisher into backtest execution pipeline**

Geänderte Dateien (13):
- `odin-app/.../BacktestExecutionService.java` — +128 Zeilen: Publisher-Erstellung, Progress-SSE, Lifecycle-Events
- `odin-backtest/.../BacktestProgress.java` — +32 Zeilen: neue Felder (dayTrades, dayPnl, cumulativePnl, maxDrawdown, instruments)
- `odin-backtest/.../BacktestRunner.java` — +84 Zeilen: neue run()-Overloads mit MonitoringEventPublisher, Propagation an PipelineFactory
- `odin-core/.../PipelineFactory.java` — +25 Zeilen: setMonitoringEventPublisher(), Übergabe an TradingPipeline
- `odin-core/.../TradingPipeline.java` — +9 Zeilen: Konstruktor-Parameter + Feld (plumbing only, nicht verwendet)
- `odin-app/.../BacktestSseWiringTest.java` — +151 Zeilen: 7 neue Wiring-Unit-Tests
- `odin-app/.../BacktestExecutionServiceTest.java` — +54 Zeilen (geändert): Tests angepasst für neue 7-Argument-run()-Signatur
- `odin-app/.../BacktestControllerTest.java` — pre-existing fix: FIVE_MINUTES → ONE_MINUTE
- `odin-core/.../ReEntryEventJsonTest.java` — TradingPipeline-Konstruktor-Update
- `odin-core/.../ReEntryWiringIntegrationTest.java` — TradingPipeline-Konstruktor-Update
- `odin-core/.../SoftBlockReEvalIntegrationTest.java` — TradingPipeline-Konstruktor-Update
- `odin-core/.../TradingPipelineTest.java` — TradingPipeline-Konstruktor-Update
- `odin-core/.../DegradationManagerIntegrationTest.java` — TradingPipeline-Konstruktor-Update

---

## Implementierungs-Entscheidungen (abweichend von Story-Spezifikation)

### Entscheidung: SimulationRunner nicht modifiziert

**Story-Spezifikation (§2):** "SimulationRunner: Neuer Konstruktor-Parameter: `MonitoringEventPublisher eventPublisher` (nullable)"

**Tatsächliche Implementierung:** SimulationRunner bleibt unverändert. Publisher wird direkt im `BacktestRunner.runSingleDay()` über `PipelineFactory.setMonitoringEventPublisher()` gesetzt, bevor der SimulationRunner für den Tag konstruiert wird.

**Wiring-Pfad (implementiert):**
```
BacktestRunner.runSingleDay()
  → new PipelineFactory(...)
  → pipelineFactory.setMonitoringEventPublisher(eventPublisher)
  → new LifecycleManager(pipelineFactory, ...)
  → new SimulationRunner(lifecycleManager, ...)
  → runner.run(tradingDate, instruments, runId)
    → pipelineFactory.createPipeline(instrumentId, runContext)
      → new TradingPipeline(..., monitoringEventPublisher)
```

**Begründung:** Sauberer Ansatz — vermeidet unnötigen Parameter durch SimulationRunner. Ergebnis (Publisher in TradingPipeline) ist identisch.

### Entscheidung: TradingPipeline nutzt Publisher noch nicht (plumbing only)

**Story-Prämisse:** "die Pipeline nutzt bereits MonitoringEventPublisher — muss nur injected werden"

**Tatsache:** TradingPipeline hat MonitoringEventPublisher als Feld, ruft es aber nie auf. Pipeline-Level-Events (indicator-update, trade-executed etc.) fließen im Backtest **nicht** über SSE. Nur Day-Summary und Bar-Progress (vom Progress-Callback) werden gesendet.

**Konsequenz:** Offene Schuld — Folge-Story erforderlich.

---

## QA-Befunde (R1)

| Severity | Befund | Aktion |
|----------|--------|--------|
| MEDIUM | SimulationRunner nicht modifiziert (Story-Abweichung) | Story-Korrektur (see open items) |
| MEDIUM | TradingPipeline: Publisher stored but unused — pipeline-level events nicht gesendet | Folge-Story anlegen |
| MEDIUM | Kein Failsafe-IntegrationTest (DoD-Anforderung) | BacktestSseWiringIntegrationTest anlegen |
| LOW | winRate-Berechnung ist Näherung (immer 0.0 oder 1.0) | In Folge-Story adressieren |
| LOW | BacktestBarProgressEvent: completedBars=0, totalBars=0 | In Folge-Story adressieren |
| LOW | BacktestRunner Singleton: cancelled-Flag nicht isoliert bei concurrent Backtests | Pre-existing, separates Ticket |

---

## Offene Punkte

1. **Story-Korrektur:** `story.md` ergänzen mit tatsächlichem Implementierungsansatz (PipelineFactory-Setter statt SimulationRunner-Parameter)
2. **Folge-Story:** "TradingPipeline nutzt MonitoringEventPublisher" — pipeline-level SSE-Events im Backtest
3. **IntegrationTest:** `BacktestSseWiringIntegrationTest` mit Spring-Context
4. **BacktestProgress:** `dayWins`/`dayLosses` für korrekte Win-Rate
5. **BacktestProgress:** `completedBars`/`totalBars` für echtes Bar-Level-Progress

---

## Test-Ergebnisse

```
BUILD SUCCESS (mvn clean install -DskipTests)
BUILD SUCCESS (mvn test)
Failures: 0, Errors: 0, Skipped: 0

ODIN-114-relevante Tests:
BacktestSseWiringTest               7/7  PASS
BacktestExecutionServiceTest       14/14 PASS
BacktestEventPublisherTest         17/17 PASS
ThrottlingEventFilterTest          24/24 PASS
Gesamt (alle Module):             ALL   PASS (0 Failures)
```

---

## Definition of Done — Abschluss-Check

| DoD-Punkt | Status | Bemerkung |
|-----------|--------|-----------|
| Code kompiliert fehlerfrei | ✓ PASS | BUILD SUCCESS |
| Alle Akzeptanzkriterien erfüllt | ⚠ CONDITIONAL | AC4 abweichend, AC8 fehlt |
| Unit-Tests (Surefire: `*Test`) | ✓ PASS | 7 neue Tests in BacktestSseWiringTest |
| Integrationstests (Failsafe: `*IntegrationTest`) | ✗ FEHLT | Kein IntegrationTest geliefert |
| ChatGPT-Sparring für Test-Edge-Cases | — | Nicht geprüft (QA-Scope) |
| Gemini-Review 3 Dimensionen | — | Nicht geprüft (QA-Scope) |
| Review-Findings eingearbeitet | — | N/A (erster QA-Lauf) |
| Protokolldatei (`protocol.md`) vollständig | ✓ PASS | Dieses Dokument |
| Commit & Push | ✓ PASS | `cd3ec52` committed und pushed |
