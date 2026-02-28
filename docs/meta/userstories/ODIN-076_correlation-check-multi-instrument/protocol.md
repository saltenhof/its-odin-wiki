# Protokoll: ODIN-076 — Correlation Check Multi-Instrument

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

### Nur positive Korrelation ausgewertet
Die Story-Notizen empfehlen, nur positive Korrelation als Risiko zu werten. ODIN ist Long-Only, daher ist negative Korrelation diversifizierend und wuenschenswert. Implementierung: `maxCorrelation` wird mit 0.0 initialisiert, nur positive Werte werden beruecksichtigt.

### Ring-Buffer fuer Return-Serien
Returns werden in `LinkedList` gespeichert mit Size-Cap = `windowBars` (Default 30). Beim Hinzufuegen wird der aelteste Eintrag entfernt wenn die Kapazitaet erreicht ist. Einfacher als `ArrayDeque` weil `subList()` fuer die Korrelationsberechnung benoetigt wird.

### Overlap-basierte Korrelationsberechnung
Wenn zwei Instrumente unterschiedlich viele Returns haben, wird der kuerzere Ueberlapp verwendet (letzte N Eintraege beider Serien). Dies stellt sicher, dass die zeitlich juengsten Daten verglichen werden.

### NaN/Infinity-Filterung
Non-finite Returns werden in `recordReturn()` abgewiesen (mit Warning-Log). Dies verhindert, dass korrupte Upstream-Daten die Korrelationsberechnung verfaelschen.

### Variance-Epsilon statt exakter Null-Vergleich
`VARIANCE_EPSILON = 1e-14` wird verwendet statt `varianceA == 0.0`, da Floating-Point-Arithmetik bei konstanten Serien (z.B. alle Returns = 0.01) nicht exakt Null produziert. Ohne Epsilon wuerde die Korrelation fuer effektiv konstante Serien nicht als NaN erkannt.

## Offene Punkte

### Zeitstempel-basierte Alignment (Gemini Dim 1)
Gemini weist darauf hin, dass die Return-Serien ohne Zeitstempel gespeichert werden. Bei Trading-Halts koennten die Serien zeitlich desynchronisieren. Fuer V1 akzeptabel, da ODIN nur 2-3 liquide US-Tech-Aktien handelt, die synchrone 5m-Bars erhalten. Fuer zukuenftige Erweiterung auf illiquide Maerkte sollte Timestamp-Alignment nachgeruestet werden.

### Start-of-Day ohne Korrelationsdaten (Gemini Dim 3)
In den ersten ~50 Minuten RTH (10 Bars x 5 Min) ist keine Korrelation berechenbar. Dies ist by Design: die Story-Notizen sagen explizit "konservativ (lieber handeln als unberechtigt blockieren)". Der Trading-Start wird nicht kuenstlich verzoegert.

## ChatGPT-Sparring

### Vorgeschlagene Szenarien und Bewertung

| # | Vorschlag | Bewertet | Umgesetzt |
|---|-----------|----------|-----------|
| 1 | NaN/Infinity in Input-Daten | Ja — valider Edge Case | Ja — NaN-Filterung in `recordReturn()` + Test |
| 2 | Negative Korrelation soll kein Risiko sein | Ja — explizit im Konzept | Ja — Test `negativeCorrelationShouldNotTriggerReduction` |
| 3 | Symmetrie-Test corr(A,B)==corr(B,A) | Ja — fundamentale Eigenschaft | Ja — Test `correlationShouldBeSymmetric` |
| 4 | Threshold-Boundary-Test (exakt auf Schwelle) | Ja — aber schwierig zu implementieren ohne gerechnete Testdaten | Nein — indirekt getestet durch "above reduce" und "above block" Tests |
| 5 | Overlap-Slicing nutzt neueste Daten | Ja — implizit getestet durch Integration | Nein — kein separater Test, aber durch Designentscheidung abgedeckt |
| 6 | Window-Trimming beeinflusst Ergebnis | Ja — wichtig fuer Ring-Buffer | Nein — Test `returnWindowShouldBeCappedAtConfiguredSize` prueft indirekt |
| 7 | `minDataPoints=0` Konfigurationsfehler | Ja — aber durch `@Min(2)` Validation verhindert | Nein — Framework-Validation genuegt |

### ChatGPT-Gesamtbewertung
Sehr konstruktives Feedback. Die wichtigsten Vorschlaege (NaN-Handling, negative Korrelation, Symmetrie) wurden umgesetzt. Threshold-Boundary und Window-Trimming sind durch die bestehenden Tests indirekt abgedeckt.

## Gemini-Review

### Dimension 1: Code-Review
| Finding | Bewertung | Aktion |
|---------|-----------|--------|
| Time-Series-Misalignment ohne Timestamps | Valider Hinweis, aber akzeptabel fuer V1 (2-3 liquide Instrumente) | Notiert als Offener Punkt |
| Concurrency-Bottleneck durch `synchronized` | Akzeptabel fuer max 3 Pipelines | Keine Aktion |
| Mathematische Stabilitaet bestaetigt | Positiv | Keine Aktion |
| Hybrid-State in AccountRiskState | By Design laut Story-Spezifikation | Keine Aktion |

### Dimension 2: Konzepttreue
Alle Akzeptanzkriterien als erfuellt bestaetigt. Keine Abweichungen gefunden.

### Dimension 3: Praxis-Review
| Finding | Bewertung | Aktion |
|---------|-----------|--------|
| Stale-Data-Risiko bei Trading-Halts | Out of scope, da nur liquide US-Tech gehandelt wird | Keine Aktion |
| Start-of-Day Blindheit | By Design (Story-Notiz: "konservativ") | Keine Aktion |
| Memory-Leak bei ungenutzten Instrumenten | Trivial bei max 3 Instrumenten, daily reset | Keine Aktion |
| Sizing-Multiplier-Compounding | By Design: hoechste paarweise Korrelation bestimmt | Keine Aktion |

### Gemini-Gesamtbewertung
Gruendliches Review ohne kritische Bugs. Die Findings betreffen hauptsaechlich Skalierbarkeits- und Erweiterungsaspekte, die fuer V1 nicht relevant sind.

## Implementierte Dateien

### Neue Dateien
- `odin-core/.../service/CorrelationChecker.java` — Pearson-Korrelationsberechnung
- `odin-core/.../service/CorrelationCheckerTest.java` — 14 Unit-Tests
- `odin-core/.../service/CorrelationCheckIntegrationTest.java` — 6 Integrationstests

### Geaenderte Dateien
- `odin-api/.../dto/AccountRiskState.java` — 2 neue Felder: `correlationRiskElevated`, `correlationSizeMultiplier`
- `odin-core/.../config/CoreProperties.java` — Neues `CorrelationProperties` Record in `GlobalRiskProperties`
- `odin-core/.../service/GlobalRiskManager.java` — Return-Recording, Korrelationsberechnung, erweiterte AccountRiskState-Erstellung
- `odin-core/src/main/resources/odin-core.properties` — Correlation-Defaults

### Angepasste Tests (Constructor-Updates)
- `odin-api/.../CycleTrackingIntegrationTest.java`
- `odin-core/.../GlobalRiskManagerTest.java`
- `odin-core/.../KillSwitchServiceIntegrationTest.java`
- `odin-core/.../DailyPerformanceRecorderIntegrationTest.java`
- `odin-core/.../TradingPipelineTest.java`
- `odin-execution/.../RiskGateTest.java`
- `odin-execution/.../RiskGatePreTradeIntegrationTest.java`
- `odin-execution/.../RiskGatePositionSizerIntegrationTest.java`
- `odin-execution/.../RiskGateHmacIntegrationTest.java`
- `odin-app/.../TradingRunControllerTest.java`
- `odin-app/.../ExchangeControllerTest.java`
- `odin-backtest/.../BacktestPipelineIntegrationTest.java`
- `odin-backtest/.../WalkForwardRunnerIntegrationTest.java`
- `odin-backtest/.../BacktestRunnerTest.java`
- `odin-backtest/.../BacktestRunnerGovernanceIntegrationTest.java`

## Test-Ergebnisse

### Unit-Tests (Surefire)
- `CorrelationCheckerTest`: 14 Tests, 0 Failures
- `GlobalRiskManagerTest`: 31 Tests, 0 Failures (inkl. 10 neue Korrelationstests)

### Integrationstests (Failsafe)
- `CorrelationCheckIntegrationTest`: 6 Tests, 0 Failures

### Gesamter Build
- `odin-api`: alle Tests OK
- `odin-core`: alle Tests OK (Unit + Integration)
- `odin-execution`: Unit-Tests OK, 1 pre-existierender Integrationstestfehler (TrailingStopManagerIntegrationTest — Locale-Problem, nicht von dieser Story)
