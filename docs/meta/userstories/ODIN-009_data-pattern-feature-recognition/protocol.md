# ODIN-009 Protocol — Pattern Feature Recognition

**Story:** ODIN-009 Data-Pattern-Feature-Recognition
**Modul:** odin-data
**Status:** Abgeschlossen

---

## Working State

| Meilenstein | Status |
|-------------|--------|
| Implementation (6 Features) | DONE |
| Compilation fix (MarketSnapshot arity) | DONE |
| Unit Tests (PatternFeatureCalculatorTest) | DONE — 33 Tests, alle GREEN |
| Integration Tests (PatternFeatureIntegrationTest) | DONE — 4 Tests, alle GREEN |
| Gemini Review | DONE |
| ChatGPT Sparring | DONE |
| Gesamtbuild odin-data | GREEN (237 Tests) |
| Gesamtbuild odin-brain + odin-execution | GREEN |
| Commit + Push | DONE |

---

## Neue Dateien

| Datei | Typ | Beschreibung |
|-------|-----|--------------|
| `odin-api/.../PatternFeatures.java` | Record | 6-Felder-DTO für Pattern-Signale |
| `odin-data/.../PatternFeatureCalculator.java` | Service | Stateful Calculator, ~570 LOC |
| `odin-data/.../PatternFeatureCalculatorTest.java` | Test | 33 Unit-Tests (Surefire) |
| `odin-data/.../PatternFeatureIntegrationTest.java` | Test | 4 Integrationstests (Failsafe-Naming, via Surefire) |

## Geänderte Dateien

- `MarketSnapshot.java` — Neues Feld `PatternFeatures patternFeatures` (20. Parameter)
- `MarketSnapshotFactory.java` — `patternFeatures`-Parameter in `createSnapshot()`
- `DataPipelineService.java` — `PatternFeatureCalculator` instanziiert und in `onBarClose()` aufgerufen
- 15+ Test- und Produktionsdateien — `PatternFeatures.empty()` als letztes Argument zu allen `new MarketSnapshot(...)` Aufrufen hinzugefügt
- `RollingDataBuffer.java` — Korrektur eines versehentlich eingeführten falschen Klassennamens (`dirRollingDataBuffer` → `RollingDataBuffer`)

---

## Design-Entscheidungen

### Flush-Start-Referenz: Window High statt Close des ältesten Bars

Ursprünglich wurde `bars.get(priorIndex).close()` als `flushStartPrice` verwendet. Nach Gemini-Review wurde dies auf `windowHigh` (das Hoch des Lookback-Fensters) geändert. Begründung: Das echte Flush-Startlevel ist die höchste Kursebene im Flush-Fenster — der Preis muss diese Level überwinden, um einen Reclaim zu bestätigen.

### Multi-Bar Flush: kein `closeNearLow`-Check

Der `closeNearLow`-Check (Close im untersten 20%-Band der Bar-Range) ist nur für den Single-Bar-Flush sinnvoll. Beim Multi-Bar-Flush über 2–5 Bars verteilt das Verkaufsdruck sich über mehrere Bars. Die Anforderung, dass JEDE Bar close-near-low schließt, wäre zu restriktiv. Multi-Bar-Flush erfordert nur: totaler Drop > 2×ATR UND letzter Bar bearish. Dies ist absichtlich und im JavaDoc dokumentiert.

### Swing-Low-Detection: Nur der aktuell bestätigbare Index wird per Call geprüft

`evaluateHigherLow` prüft pro `update()`-Aufruf nur den Index `bars.size() - 1 - SWING_LOW_LOOKBACK`. Konsequenz: Der Calculator muss **inkrementell** (Bar für Bar) aufgerufen werden. Ein nachträglicher Batch-Aufruf mit 100 Bars würde nur den 97. Bar auf Swing-Low prüfen. Dies entspricht der intended Usage im `DataPipelineService` (einmal pro `onBarClose()`). In den Tests wurde dies durch inkrementelle Build-Up-Schleifen dokumentiert.

### Reclaim-Kondition: flushStartPrice = windowHigh AND price > VWAP

Konzept sagt "Preis > MA-Cluster". Da EMA nicht in odin-data verfügbar ist, wurde VWAP als Proxy für die Bestätigung verwendet, zusammen mit dem windowHigh als stationäres Reclaim-Niveau. Dies ist ein pragmatischer Kompromiss (in JavaDoc dokumentiert). Die Pattern-FSMs in ODIN-012/013 können die vollständige MA-Cluster-Prüfung dann in odin-brain ergänzen.

### Warm-up: 14 Bars minimum (ATR_PERIOD)

`update()` gibt `PatternFeatures.empty()` zurück wenn `bars.size() < 14`. Einzelne Features (Coiling, Breakout) benötigen intern 20 Bars und geben bis dahin 0.0 zurück. Dies ist dokumentiertes Verhalten, kein Bug.

### Unbounded `confirmedSwingLows`

Die Liste wächst über die Session. Bei ~390 1-Minuten-Bars in einer RTH-Session (6.5h) ist dies vernachlässigbar. Ein Cap wäre nur bei sehr langen Sessions relevant und würde das Pullback-Depth-Verhalten verändern. Bleibt vorerst unbegrenzt.

---

## Gemini Review — Findings und Bewertung

**Dimension 1: Bugs**

| Finding | Bewertet | Aktion |
|---------|----------|--------|
| Flush-Start-Preis war `close[priorIndex]`, nicht `windowHigh` | **Berechtigt** | Gefixt: `flushStartPrice = windowHigh` |
| `confirmedSwingLows` wächst unbegrenzt | Geringes Risiko bei Intraday | Dokumentiert, nicht gefixt |
| Init-Bug bei Pre-Warmed Buffer | Design-Entscheidung (inkrementell) | JavaDoc ergänzt |

**Dimension 2: Konzepttreue**

| Finding | Bewertet | Aktion |
|---------|----------|--------|
| Multi-Bar Flush ignoriert `closeNearLow` | Absichtlich | JavaDoc-Erklärung ergänzt |
| Reclaim: `flushStartPrice` statt MA-Cluster | Pragmatischer Kompromiss | JavaDoc-Verweis bereits vorhanden |
| Features sind reine Inputs (keine Entscheidungen) | Bestätigt ✓ | Keine Aktion |

**Dimension 3: Praxis-Gaps**

| Finding | Bewertet | Aktion |
|---------|----------|--------|
| Warm-up: Coiling/Breakout brauchen 20 Bars, guard ist bei 14 | Bewusste Entscheidung | Dokumentiert |
| Opening-Gap verzerrt ATR + BB | Wichtige Beobachtung | Offener Punkt in ODIN-Wiki |
| Swing-Low Ties (Double Bottom = false) | Standard in der TA | Test hinzugefügt |

---

## ChatGPT Sparring — Edge Cases

ChatGPT hat folgende Edge Cases identifiziert (Auswahl):

| Edge Case | Ergebnis | Test |
|-----------|----------|------|
| Flush-Drop exakt 2×ATR (nicht >) | Kein Flush (strict >) | `flushNotDetected_exactlyTwoTimesAtrDrop` |
| ATR=0 übergeben → kein Crash | Graceful degradation | `flushNotDetected_atrIsZero` |
| Baseline alle Zero-Volume → BreakoutStrength | 0.0 (division guard) | `breakoutStrength_zeroWhenAllBaselineZeroVolume` |
| PullbackDepth wenn upMove < 0.5×ATR | 0.0 | `pullbackDepth_zeroWhenUpMoveBelowMinimum` |
| Double Bottom (gleiche Swing-Low-Levels) | higherLowConfirmed=false | `higherLowNotConfirmed_doubleBotom` |

Nicht implementierte Edge Cases (dokumentiert, kein Test):
- Coiling-Score genau an der Schwelle (0.02): Code nutzt `<`, Gleichheit resetet. Aufwand vs. Nutzen gering.
- Reclaim auf dem letzten Bar im Fenster: Bereits durch `reclaimDetected_afterFlushPriceAboveStartAndVwap` implizit abgedeckt.

---

## QS-Abschluss

| Runde | Datum | Ergebnis | Report |
|-------|-------|---------|--------|
| R1 | 2026-02-21 | PASS | `qa-report-r1.md` |

6 Findings (3x MINOR, 3x INFO), alle als tolerierbar oder bewusste Design-Entscheidungen bewertet. Keine Blocker.

---

## Offene Punkte (Eskalation)

1. **Opening-Gap-Distortion**: Kombination von Pre-Market-Daten und RTH-Daten verzerrt ATR und Bollinger Bandwidth in den ersten 20 Minuten. Wenn ODIN in den ersten 20 Minuten nach Open tradet, können die Pattern-Features irreführend hohe Breakout-Strength oder falsche Flush-Signale liefern. Vorschlag: Calculator-Reset bei RTH-Open oder Session-Marker-basiertes Handling (ODIN-016 / Backlog).

2. **Historical Replay-Modus**: Falls `DataPipelineService` jemals mit einem Pre-Filled Buffer von historischen Bars startet (Warm-Start nach Crash-Recovery), würden Swing-Lows aus dem "vorherigen" Puffer nicht korrekt erkannt. Aktuell kein Use Case, aber für Crash-Recovery relevant (ODIN-010/Backlog).
