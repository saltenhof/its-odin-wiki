# Protokoll: ODIN-065 — Intraday Seasonality Normalization

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (56+2 neue Tests in Runde 1, 2 Akzeptanzkriterien-Tests in Runde 2)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (SeasonalityGateIntegrationTest, 4 Tests, Failsafe)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet (P0 Slot-Index-Fix, NaN-Volatility-Filter, Akzeptanzkriterien-Tests)
- [x] Produktionsverdrahtung (PipelineFactory + BacktestRunner)
- [x] Commit & Push

## Design-Entscheidungen

### Cross-Module Dependency Resolution
- Problem: KpiEngine (odin-brain) brauchte IntradaySeasonalityService (odin-data), aber Fachmodule duerfen sich nicht gegenseitig importieren.
- Loesung: Slot-Index-Berechnung und Surprise-Computation nach ExpectedCurve in odin-api verschoben. IntradaySeasonalityService und SeasonalitySlot delegieren nun an ExpectedCurve.

### Slot-Index-Berechnung mit openTime (nicht closeTime)
- Urspruenglich wurde bar.closeTime() fuer Slot-Zuweisung verwendet.
- Problem (ChatGPT P0 Finding): 09:35 close = Slot 1 (statt Slot 0), 16:00 close = Slot -1 (letzter Bar wird gedroppt).
- Fix: computeCurves() verwendet nun bar.openTime(). Bars werden korrekt ihrem Start-Slot zugewiesen.

### Fallback-Verhalten der Gates
- VolumeGate: Bevorzugt volumeSurprise wenn finite, sonst Fallback auf volumeRatio.
- AtrGate: Bevorzugt volatilitySurprise wenn finite, sonst Fallback auf atrDecayRatio.
- NaN effective ratio = Gate FAIL (defensives Verhalten).

### Produktionsverdrahtung (Runde 2)
- PipelineFactory: setExpectedCurve() Methode + factoryExpectedCurve Feld. createPipeline() injiziert Curve in KpiEngine wenn vorhanden.
- BacktestRunner: computeSeasonalityCurve() laedt 5-Min-Bars aus der DB via IntradayBarJdbcRepository, berechnet ExpectedCurve, setzt sie auf PipelineFactory.
- Live-Modus: Verdrahtung noch nicht implementiert (erfordert DB-Zugriff auf historische Bars vor RTH-Open, eigene Story).

## Test-Ergebnisse

### Runde 1
- ExpectedCurveTest: 24 Tests PASS
- IntradaySeasonalityServiceTest: 13 Tests PASS
- SeasonalitySlotTest: 9 Tests PASS
- VolumeGateTest: 5 Tests PASS
- AtrGateTest: 5 Tests PASS

### Runde 2 (zusaetzlich)
- VolumeGateTest: 7 Tests PASS (+2 Akzeptanzkriterien-Tests: Midday-Slot, Opening-Slot)
- SeasonalityGateIntegrationTest: 4 Tests PASS (End-to-End Curve Computation -> Gate Evaluation)
- Alle bestehenden Tests: PASS

## ChatGPT-Sparring

### Durchgefuehrt: 2026-02-28 (Runde 1)

### P0 Findings (behoben)
- **Slot-Index mit closeTime**: Letzter Bar (16:00 close) wurde gedroppt. Fix: openTime verwenden.

### P1 Findings (akzeptiert/deferred)
- **Per-Metric Availability**: isAvailable() prueft beide Maps non-empty. Theoretisch koennten Volume/Volatility unabhaengig verfuegbar sein. In der aktuellen Implementierung werden immer beide zusammen befuellt, kein Risiko aktuell.
- **Duplicate Bars Bias**: Mehrere Bars pro (day, slot) koennen Median verzerren. Bei sauberen Bar-Daten kein Problem.
- **lookbackDays nicht enforced**: Caller ist verantwortlich fuer korrekte Datenmenge. By design.
- **Bad Volatility Inputs (high < low)**: Nicht gefiltert. Daten-Validierung gehoert in die Data-Pipeline, nicht in die Seasonality-Berechnung.

### Zusaetzliche Tests hinzugefuegt nach Sparring
- computeCurves_lastSlot_1555open_included (Slot 77 = letzter Bar)
- computeCurves_multipleSlots_correctlyBucketed (Slot 0 und 1 korrekt zugewiesen)
- computeVolumeSurprise_infinityActual_returnsInfinity
- computeVolumeSurprise_nanActual_returnsNaN

## Gemini-Review

### Durchgefuehrt: 2026-02-28 (Runde 2)

### Dimension 1: Code-Review (Bugs, Race Conditions, Null-Safety, Performance)

| Severity | Finding | Bewertung |
|----------|---------|-----------|
| P1 | KpiEngine.reset() nullt expectedCurve — bei Pipeline-Wiederverwendung ueber Tage hinweg wuerde Seasonality nach Tag 1 deaktiviert | Akzeptiert: Pipelines sind per-Day POJOs, werden NICHT wiederverwendet. reset() ist ein Safety-Net, kein Produktions-Flow. |
| P2 | Fehlender NaN-Filter fuer Volatility in computeCurves() — korrupte high/low-Werte koennten NaN in die Median-Berechnung einschleusen | **BEHOBEN**: Defensiver Check `!Double.isNaN(volatility) && volatility > 0.0` hinzugefuegt |
| P3 | Object-Allocation in Loop (toLocalDate() pro Bar) | Akzeptiert: Pre-RTH Berechnung, nicht auf dem kritischen Pfad |

### Dimension 2: Konzepttreue-Review

| Severity | Finding | Bewertung |
|----------|---------|-----------|
| P0 | Alle Kern-Requirements umgesetzt: 78 Slots, Median, volumeSurprise/volatilitySurprise, Fallback | PASS |
| P2 | Flat Bars (volatility=0) erzeugen volatilitySurprise=0.0, was AtrGate (threshold >0.5) FAILen laesst | Korrektes defensives Verhalten: Ein komplett flacher Bar hat keine handelbare Volatilitaet |

### Dimension 3: Praxis-Review (reale Marktbedingungen)

| Severity | Finding | Bewertung |
|----------|---------|-----------|
| P1 | Early Closures (Half-Days, z.B. Tag nach Thanksgiving um 13:00 ET) — Nachmittags-Slots haetten stale/leere Curves | Dokumentiert als offener Punkt. Out-of-Scope fuer ODIN-065 (erfordert SessionCalendar-Integration). |
| P2 | Holiday Lookback Dilution — Feiertage reduzieren unique-day-Count | Gehandelt durch minDaysForCurve Safety Net |
| P3 | DST-Transitions | Korrekt gehandelt via ZoneId("America/New_York") |

## Offene Punkte
- **Live-Modus Verdrahtung**: Seasonality-Curves werden aktuell nur im Backtest-Modus berechnet und injiziert. Fuer den Live-Betrieb muss die Kurvenberechnung in den LifecycleManager oder einen Pre-RTH-Scheduler integriert werden (erfordert DB-Zugriff auf historische 5-Min-Bars). Eigene Story.
- **Half-Day Sessions**: Die statische RTH-Definition (09:30-16:00 ET) deckt verkuerzte Handelstage nicht ab. Fuer Early Closures wuerde ein SessionCalendar benoetigt, der den tatsaechlichen Handelsschluss kennt. Eigene Story.
- **P1 Findings aus Runde 1 (ChatGPT)**: Per-Metric Availability, Duplicate Bars Bias — bewusst akzeptiert, kein aktuelles Risiko.
