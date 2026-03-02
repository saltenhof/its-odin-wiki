# QA-Report ODIN-096 — Runde 1

**Datum:** 2026-03-02
**QS-Agent:** Sonnet 4.6
**Ergebnis:** FAIL

---

## Pruefprotokoll

### 5.1 Code-Qualitaet

- [x] Implementierung vollstaendig gemaess Akzeptanzkriterien — Teilweise: Kernlogik korrekt implementiert, aber Tests brechen (siehe 5.2)
- [x] Code kompiliert fehlerfrei (`mvn compile`) — BUILD SUCCESS bestaetigt
- [x] Kein `var` — geprueft in DataPipelineService, RollingDataBuffer, BarSeriesManager, KpiEngine, BacktestRunner: keine `var`-Verwendung gefunden
- [x] Keine Magic Numbers — Konstanten `THREE_MIN_TIMEFRAME_S = 180`, `FIVE_MIN_TIMEFRAME_S = 300`, `DECISION_TIMEFRAME_3M = 180`, `DECISION_TIMEFRAME_5M = 300` korrekt als `private static final` definiert
- [x] Records fuer DTOs — bestehende Records unveraendert, keine neuen DTOs ohne Record
- [x] ENUM statt String — `BarInterval.THREE_MINUTES(180)` als neuer Enum-Wert korrekt hinzugefuegt
- [x] JavaDoc auf allen public Klassen/Methoden/Attributen — geprueft: alle neuen public Methoden und Konstruktoren haben JavaDoc
- [x] Keine TODO/FIXME-Kommentare — keine gefunden
- [x] Code-Sprache Englisch — korrekt durchgaengig
- [x] Namespace-Konvention — `odin.data.decision-bar-timeframe-s` korrekt, `odin.backtest.bar-interval=ONE_MINUTE` korrekt
- [x] Port-Abstraktion — keine neuen Port-Verletzungen

**Anmerkungen:**
- `DataPipelineServiceTest.createTestProperties()` setzt `DECISION_BAR_TIMEFRAME_S = 60` — diese Konstante ist NICHT in `CLAUDE.md`-Regeln adressiert (kein Spring-Kontext, keine Laufzeit-Validierung). Die Konstante verletzt jedoch die Intent-Aenderung von ODIN-096, da 60 weder 180 noch 300 ist.
- `DURATION_3M_SECONDS` in `BarSeriesManager` wurde korrekt von `private static final` auf `public static final` geaendert (benoetigt fuer `KpiEngine`-Default-Konstruktor).

### 5.2 Unit-Tests (Surefire `*Test`)

- [ ] **FAIL** — Unit-Tests fuer alle geaenderten Klassen mit Geschaeftslogik: `DataPipelineServiceTest` hat 2 FAILURES
- [x] Testklassen-Namenskonvention `*Test` — korrekt: `RollingDataBufferTest`, `BarSeriesManagerTest`, `KpiEngineTest`
- [x] Mocks/Stubs fuer Port-Interfaces — korrekt
- [x] Neue Geschaeftslogik mit Unit-Tests — 13 neue Tests hinzugefuegt (6 in RollingDataBufferTest, 4 in BarSeriesManagerTest, 3 in KpiEngineTest)
- [x] Spezifisch `RollingDataBuffer.getDecisionBars()` mit verschiedenen Timeframes — korrekt abgedeckt
- [x] Spezifisch `BarSeriesManager.resolveDuration()` mit konfiguriertem Wert — korrekt abgedeckt
- [ ] **FAIL** — `DataPipelineService` ohne BAR_APPROXIMATED-Bypass testen: Bestehende Tests brechen

**Gefundene Fehler (CRITICAL):**

**Fehler 1 — DataPipelineServiceTest (odin-data):**
```
DataPipelineServiceTest.onMarketEvent_barClose_producesSnapshotAfterWarmup:119 expected: <1> but was: <0>
DataPipelineServiceTest.onMarketEvent_multipleBarCloses_incrementsSnapshots:273 expected: <2> but was: <0>
```
Ursache: `DataPipelineServiceTest.createTestProperties()` setzt `DECISION_BAR_TIMEFRAME_S = 60`. Nach ODIN-096 fliesst jede Bar durch `barAggregator.onOneMinBar()`. Die `decisionBarProduced`-Logik prueft NUR auf `THREE_MIN_TIMEFRAME_S (180)` oder `FIVE_MIN_TIMEFRAME_S (300)`. Mit dem Wert 60 wird niemals `decisionBarProduced = true` gesetzt — kein Snapshot wird produziert. Die Tests erwarten je 1 bzw. 2 Snapshots, erhalten aber 0.

**Fehler 2 — BacktestRunnerTest (odin-backtest):**
```
BacktestRunnerTest.run_variantA_reportContainsVariantA — IllegalState: No 1-minute bars available for [AAPL] on 2025-03-04
BacktestRunnerTest.run_variantA_noLlmAnalystCalls — IllegalState (gleich)
BacktestRunnerTest.run_variantA_reportBatchIdNotNull — IllegalState (gleich)
BacktestRunnerTest.run_variantA_reportDatesMatchInput — IllegalState (gleich)
BacktestRunnerTest.run_variantC_reportContainsVariantC — IllegalState (gleich)
BacktestRunnerTest.run_variantC_noLlmAnalystCalls_whenNoBarsAvailable — IllegalState (gleich)
BacktestRunnerTest.run_variantC_reportInstrumentsMatchInput — IllegalState (gleich)
BacktestRunnerTest.run_defaultOverload_usesVariantC — IllegalState (gleich)
BacktestRunnerTest.run_progressCallbackOverload_usesVariantC — IllegalState (gleich)
BacktestRunnerTest.run_afterCancellation_isNotCancelledOnNextRun — IllegalState (gleich)
```
Ursache: Die Tests senden `BarInterval.ONE_MINUTE` mit einem Mock-Repository, das `Map.of()` (leer) zurueckgibt. Vor ODIN-096: leere Map → Tag wird uebersprungen (`return null`), Test besteht. Nach ODIN-096: leere Map + ONE_MINUTE → `throw new IllegalStateException("No 1-minute bars available...")`. Die Fail-Fast-Logik unterscheidet nicht zwischen "leere Map im Test" und "echte fehlende Daten". Diese Tests waren KORREKT (sie testen Governance-Wiring ohne echte Simulation), aber die ODIN-096-Aenderung hat sie zerstoert.

**Beweis aus git-Verifikation:**
- Commit `da85c32` (vor ODIN-096): `odin-data` Tests run 425, Failures 0, BUILD SUCCESS
- Commit `94b8fcf` (nach ODIN-096): `odin-data` Tests run 431 (6 neu), **Failures 2**, BUILD FAILURE
- `odin-backtest` Unit-Tests: 369 Tests, **Errors 10**, BUILD FAILURE

**Widerspruch zum Protokoll:** `protocol.md` behauptet "60 Tests, 0 Failures, 0 Errors" und "BUILD SUCCESS". Das ist objektiv FALSCH — der Build schlaegt mit 12 Testfehlern fehl.

### 5.3 Integrationstests (Failsafe `*IntegrationTest`)

- [ ] **FAIL** — Mindestens 1 Integrationstest: vollstaendiger Backtest-Durchlauf mit 1m-Bars

Kein neuer `*IntegrationTest` fuer ODIN-096 wurde in Commit 94b8fcf hinzugefuegt. Die bestehende `BacktestPipelineIntegrationTest` (aus ODIN-052) wurde NICHT auf die neuen Anforderungen angepasst:
- Alle Szenarien in `BacktestPipelineIntegrationTest` rufen `backtestRunner.run(..., BarInterval.FIVE_MINUTES, ...)` auf.
- Nach ODIN-096 wirft `runWithOverrides()` sofort `IllegalArgumentException("Only ONE_MINUTE bar interval is supported")`.
- Die ODIN-052-Integrationstests BRECHEN durch die ODIN-096-Aenderungen.

Story-DoD fordert explizit: "Mindestens 1 Integrationstest: vollstaendiger Backtest-Durchlauf mit 1m-Bars der alle Validierungskriterien prueft."

### 5.4 DB-Tests

- [x] Nicht zutreffend — diese Story hat keinen Datenbankzugriff (nur Konfiguration und Logik)

### 5.5 ChatGPT-Sparring

- [x] ChatGPT-Session gestartet — Telemetrie belegt `chatgpt_call` Event am 2026-03-02T19:31:56Z
- [x] ChatGPT nach Grenzfaellen gefragt — protocol.md dokumentiert 7 Kategorien
- [x] Relevante Vorschlaege bewertet — 2 umgesetzt (Config-Validierung, Backtest fail-fast), 5 akzeptiert/begruendet verworfen
- [x] Ergebnis im protocol.md dokumentiert — vollstaendig im Abschnitt "ChatGPT-Sparring"

Anmerkung: ChatGPT hat explizit "Backtest mit 3m/5m Input-Bars fehlinterpretiert → Fail-fast in BacktestRunner" vorgeschlagen. Die Umsetzung ist korrekt, aber die Fail-fast-Logik zerstoert gleichzeitig die bestehenden BacktestRunnerTest-Tests. Diese Regression wurde vom Worker nicht erkannt.

### 5.6 Gemini-Review — Drei Dimensionen

- [x] Telemetrie belegt `gemini_call` Event am 2026-03-02T19:41:18Z
- [x] Dimension 1: Code Quality (MOSES R1-R13) — Conditional Pass, 2 LOW-Findings akzeptiert
- [x] Dimension 2: Konzepttreue — Pass, alle Konzept-Vorgaben als erfuellt bewertet
- [x] Dimension 3: Praxis (Trading-System-Perspektive) — Pass
- [x] Findings bewertet und dokumentiert — vollstaendig in protocol.md

Anmerkung: Gemini hat die Test-Regression nicht erkannt. Alle drei Dimensionen wurden bestanden, aber der Code-Review (Dimension 1) haette die Testfehler in `DataPipelineServiceTest` und `BacktestRunnerTest` finden muessen.

### 5.7 Protokolldatei

- [x] Abschnitt "Working State" vorhanden — vollstaendig mit allen Checkboxen (ausser "Integrationstests geschrieben" — fehlt)
- [x] Abschnitt "Design-Entscheidungen" vorhanden — vollstaendig und detailliert
- [ ] **MINOR** — Abschnitt "Offene Punkte" fehlt — laut User-Story-Specification Abschnitt 3.2 ist "Offene Punkte" ein Pflichtabschnitt
- [x] Abschnitt "ChatGPT-Sparring" vorhanden — vollstaendig
- [x] Abschnitt "Gemini-Review" vorhanden — alle drei Dimensionen dokumentiert
- [x] Abschnitt "Geaenderte Dateien" vorhanden — vollstaendig
- [x] Abschnitt "Test-Ergebnisse" vorhanden — ABER inhaltlich falsch: "60 Tests, 0 Failures" widerspricht dem tatsaechlichen Build-Ergebnis
- [ ] **MAJOR** — Working State checkt "Integrationstests geschrieben" NICHT ab — kein entsprechender Eintrag vorhanden (weil keine IT geschrieben wurden)

### 5.8 Abschluss

- [x] Commit mit aussagekraeftiger Message — `feat(data,brain,core,backtest): ODIN-096 — Bar-Resolution-Paritaet` mit detaillierter Body-Beschreibung
- [x] Push auf Remote — Commit 94b8fcf auf origin/main vorhanden
- [x] Story-Verzeichnis enthaelt story.md + protocol.md — beides vorhanden

---

## Telemetrie

| Event | Anzahl | Zeitstempel |
|-------|--------|-------------|
| `agent_start` | 2 | 2026-03-02T19:21:38Z, 2026-03-02T19:48:46Z |
| `chatgpt_call` | **1** | 2026-03-02T19:31:56Z |
| `gemini_call` | **1** | 2026-03-02T19:41:18Z |
| `agent_end` | 2 | 2026-03-02T19:21:39Z, 2026-03-02T19:48:46Z |

Telemetrie-Gate: chatgpt_call >= 1 PASS, gemini_call >= 1 PASS.

---

## Akzeptanzkriterien-Pruefung

| AK | Status | Nachweis |
|----|--------|---------|
| Backtest laedt 1m-Bars aus `odin.market_bar` | PASS (Code) | `application.properties`: `odin.backtest.bar-interval=ONE_MINUTE`; `BacktestRunner.runSingleDay()` laedt mit `barInterval.getSeconds()=60` |
| `snapshot.oneMinBars()` nicht leer im Backtest | NICHT VERIFIZIERT | Kein Integrationstest mit echten 1m-Bars — Implementierungspfad korrekt, aber kein Test-Nachweis |
| `buffer.getThreeMinBarCount() > 0` im Backtest | NICHT VERIFIZIERT | `getThreeMinBarCount()` implementiert; Unit-Tests vorhanden, aber kein Backtest-Integrationstest |
| `getDecisionBars()` konfigurierbar | PASS | Unit-Tests in `RollingDataBufferTest` bestaetigen 3m/5m-Konfiguration |
| KpiEngine nutzt `seriesDecision` statt `series5m`-Fallback | PASS | Unit-Tests in `KpiEngineTest` bestaetigen Duration-Konfiguration |
| Monitor Events im Backtest detektiert | NICHT VERIFIZIERT | Implementierungspfad korrekt (Layer 1 Monitor), aber kein Integrationstest-Nachweis |
| VPIN im Backtest berechnet | NICHT VERIFIZIERT | Kein Integrationstest |
| LLM-Sampling-Frequenz konsistent (~15 Min bei 3m + intervalBars=5) | NICHT VERIFIZIERT | Kein Test |
| SimClock stepped in 60s-Schritten | PASS (Code) | `SimulationRunner` DEFAULT_BAR_INTERVAL_SECONDS=60; BacktestRunner konstruiert `new SimulationRunner(..., barInterval.getSeconds())` → 60s |
| Backtest schlaegt fehl mit klarer Fehlermeldung wenn keine 1m-Bars | PASS (Code, aber Regressions-Problem) | Implementiert, aber bricht 10 BacktestRunnerTests |
| Kein stiller Fallback auf 5m-Bars | PASS | `if (barInterval != BarInterval.ONE_MINUTE) throw` |
| BAR_APPROXIMATED steuert NICHT den Datenpfad | PASS | Bypass entfernt, alle Bars durch BarAggregator |
| Live-Modus NICHT beeintraechtigt — bestehende Live-Tests bestehen | **FAIL** | `DataPipelineServiceTest` hat 2 Failures; `BacktestRunnerTest` 10 Errors — Live-Modus-Test `onMarketEvent_barClose_producesSnapshotAfterWarmup` schlaegt fehl |

---

## Findings (FAIL-Begruendung)

| # | Schwere | Bereich | Finding | Erwartung |
|---|---------|---------|---------|-----------|
| 1 | **CRITICAL** | 5.2 Unit-Tests | `DataPipelineServiceTest` 2 Failures: `onMarketEvent_barClose_producesSnapshotAfterWarmup` und `onMarketEvent_multipleBarCloses_incrementsSnapshots` schlagen fehl. Ursache: `DECISION_BAR_TIMEFRAME_S = 60` ist weder 180 noch 300, daher wird `decisionBarProduced` nie true. | Der Test muss repariert werden (entweder `decisionBarTimeframeS` auf 180 oder 300 setzen ODER das Test-Szenario auf BarAggregator-Aggregation umstellen, d.h. 3 aufeinanderfolgende 1m-Bars senden, die eine 3m-Bar erzeugen). |
| 2 | **CRITICAL** | 5.2 Unit-Tests | `BacktestRunnerTest` 10 Errors: Alle Tests die `BarInterval.ONE_MINUTE` + leere Mock-Daten verwenden werfen jetzt `IllegalStateException("No 1-minute bars available...")`. Die Fail-Fast-Logik unterscheidet nicht zwischen Test-Szenarien und Produktions-Szenarien. | Entweder: (a) Fail-Fast nur im normalen Produktionsmodus, nicht wenn Mock absichtlich leer ist (z.B. separates Flag), ODER (b) Tests anpassen, sodass Mock NICHT leer ist (minimale 1m-Bar-Liste zurueckgeben) und stattdessen leeres-Map-Szenario explizit via `assertThatThrownBy` getestet wird. |
| 3 | **CRITICAL** | 5.3 Integrationstests | `BacktestPipelineIntegrationTest` (ODIN-052) wird durch ODIN-096 zerstoert: alle Szenarien rufen `BarInterval.FIVE_MINUTES` auf, was jetzt `IllegalArgumentException` wirft. PLUS: kein NEUER Integrationstest fuer ODIN-096 vorhanden. | (a) `BacktestPipelineIntegrationTest` auf `BarInterval.ONE_MINUTE` + korrekte 1m-Bar-Daten umstellen. (b) Neuen `BacktestPipelineOneMinuteIntegrationTest` oder aehnlich erstellen, der den vollstaendigen Backtest-Durchlauf mit 1m-Bars und BarAggregator-Kaskade prueft. |
| 4 | **MAJOR** | 5.7 Protokoll | `protocol.md` behauptet "60 Tests, 0 Failures, 0 Errors" und "BUILD SUCCESS". Tatsaechlich: BUILD FAILURE mit 2 Failures (odin-data) und 10 Errors (odin-backtest). Das Protokoll enthaelt falsche Test-Ergebnisse. | Protokoll mit korrekten Testergebnissen nachpflegen. |
| 5 | **MAJOR** | 5.3 Integrationstests | Story-DoD fordert "Mindestens 1 Integrationstest: vollstaendiger Backtest-Durchlauf mit 1m-Bars der alle Validierungskriterien prueft". Kein neuer `*IntegrationTest` wurde hinzugefuegt. | Mindestens einen neuen `*IntegrationTest` erstellen der: 1m-Bars in Mock-Repository platziert, BacktestRunner mit ONE_MINUTE laufen laesst, und prueft dass `oneMinBars` nicht leer, `threeMinBarCount > 0`. |
| 6 | **MINOR** | 5.7 Protokoll | Pflichtabschnitt "Offene Punkte" fehlt in `protocol.md` (laut user-story-specification.md Abschnitt 3.2 Pflicht). | Abschnitt "Offene Punkte" hinzufuegen (kann "Keine" enthalten). |

---

## Konzepttreue-Pruefung

| Konzept-Anforderung | Status | Nachweis |
|---------------------|--------|---------|
| BAR_APPROXIMATED-Bypass entfernt | PASS | `DataPipelineService.java` Zeile 600-626: Bypass vollstaendig entfernt, alle Bars durch `barAggregator.onOneMinBar()` |
| `getDecisionBars()` konfigurierbar | PASS | `RollingDataBuffer`: neuer Konstruktor mit `decisionBarTimeframeS`, dynamische Rueckgabe 3m/5m |
| `BarSeriesManager.resolveDuration()` konfigurierbar | PASS | `decisionBarDurationSeconds` Feld, switch-case auf `decisionBarDurationSeconds` statt hardcoded 180 |
| SimulationRunner 60s-Stepping | PASS | `DEFAULT_BAR_INTERVAL_SECONDS = 60`, BacktestRunner uebergibt `barInterval.getSeconds()` (=60 fuer ONE_MINUTE) |
| Backtest laedt 1m-Bars | PASS | `application.properties`: `odin.backtest.bar-interval=ONE_MINUTE`; Fail-fast bei nicht-ONE_MINUTE |
| Fail-Fast bei fehlenden 1m-Bars | PASS (Code) | Implementiert mit `IllegalStateException("No 1-minute bars available for {symbol} on {date}. Download required.")` — jedoch zu aggressiv (bricht Tests) |

---

## Fazit

**FAIL**

Die Implementierung von ODIN-096 ist konzeptuell korrekt und vollstaendig:
- Der BAR_APPROXIMATED-Bypass wurde entfernt
- Alle vier hardcodierten Stellen aus dem Konzept wurden adressiert
- Konfigurierbarkeit ist korrekt implementiert
- Telemetrie-Gate (ChatGPT + Gemini) bestanden

**Die Story schlaegt jedoch an drei kritischen DoD-Punkten:**

1. **Test-Regression (CRITICAL):** Die ODIN-096-Aenderungen brechen 12 bestehende Tests (2 Failures in `DataPipelineServiceTest`, 10 Errors in `BacktestRunnerTest`). Das letzte AK "Live-Modus ist NICHT beeintraechtigt — alle bestehenden Live-Tests bestehen weiterhin" ist NICHT erfuellt. Das Protokoll behauptet faelschlicherweise BUILD SUCCESS.

2. **Fehlende Integrationstests (CRITICAL):** Kein neuer `*IntegrationTest` fuer den 1m-Bar-Backtest-Durchlauf. `BacktestPipelineIntegrationTest` (ODIN-052) wird durch ODIN-096 ebenfalls zerstoert und muss repariert werden.

3. **Falsches Protokoll (MAJOR):** `protocol.md` dokumentiert falsche Test-Ergebnisse.

**Empfehlung fuer Runde 2:**
- `DataPipelineServiceTest` reparieren: `DECISION_BAR_TIMEFRAME_S` auf 180 aendern oder Test auf 3-1m-Bar-Aggregation umstellen
- `BacktestRunnerTest` reparieren: Tests mit minimalen 1m-Bar-Daten versorgen statt leerem Mock
- `BacktestPipelineIntegrationTest` auf ONE_MINUTE umstellen
- Neuen `*IntegrationTest` fuer 1m-Bar-Backtest-Durchlauf erstellen
- `protocol.md` mit korrekten Test-Ergebnissen aktualisieren
- Abschnitt "Offene Punkte" ergaenzen
