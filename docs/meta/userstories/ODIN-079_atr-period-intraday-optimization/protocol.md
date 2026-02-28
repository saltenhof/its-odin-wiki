# Protokoll: ODIN-079 — ATR Period Intraday Optimization

## Working State
- [x] Initiale Verifikation und JavaDoc-Ergaenzungen
- [x] Unit-Tests geschrieben (6 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (2 Tests)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Verifikationsergebnis

### Propagation Chain: VOLLSTAENDIG VERIFIZIERT

Die ATR-, RSI- und ADX-Perioden werden korrekt von der Konfiguration bis zu den ta4j-Indikatoren propagiert:

```
odin-brain.properties (odin.brain.kpi.atr-period=14, rsi-period=14, adx-period=14)
    -> BrainProperties.KpiProperties (atrPeriod, rsiPeriod, adxPeriod, alle @Min(1))
        -> KpiEngine.onSnapshot()
            -> computeRsi(series5m, kpiProperties.rsiPeriod())  -> new RSIIndicator(closePrice, period)
            -> computeAtr(series5m, kpiProperties.atrPeriod())  -> new ATRIndicator(series, period)
            -> computeAdx(series5m, kpiProperties.adxPeriod())  -> new ADXIndicator(series, period)
            -> computePlusDi(series5m, kpiProperties.adxPeriod()) -> new PlusDIIndicator(series, period)
            -> computeMinusDi(series5m, kpiProperties.adxPeriod()) -> new MinusDIIndicator(series, period)
```

**Keine hardcodierten "14" in der Berechnungskette.** Alle Perioden werden aus Properties gelesen.

### Downstream-Konsumenten: KEIN PROBLEM

Alle Konsumenten (RegimeResolver, GateCascadeEvaluator, EntryRules, ExitRules, ExhaustionDetector, PromptBuilder) greifen auf IndicatorResult-Werte zu (z.B. `result.atr14_5m()`), NICHT auf die Perioden-Konfiguration. Die Perioden-Aenderung ist transparent fuer alle Downstream-Klassen.

### IndicatorResult Feldnamen

Die Felder heissen `rsi14_5m`, `atr14_5m`, `adx14_5m` (inkl. "14" im Namen). Dies ist eine bewusste Entscheidung: Umbenennen waere ein Breaking Change fuer >90 Dateien. Stattdessen wurden JavaDoc-Kommentare ergaenzt, die klarstellen dass "14" den Default referenziert und die tatsaechliche Periode konfigurierbar ist.

## Design-Entscheidungen

1. **Kein Umbenennen der Felder**: `rsi14_5m` bleibt `rsi14_5m`. JavaDoc dokumentiert die Konfigurierbarkeit. Breaking Change vermieden.
2. **TestBrainProperties.createWithKpiPeriods()**: Neue Factory-Methode fuer Tests mit nicht-default Perioden. Ermoeglicht saubere parametrisierte Tests ohne Code-Duplizierung.
3. **Volatile Testdaten**: Sinus-Modulation + Trend-Komponente statt monotoner Preissteigerung, damit verschiedene Perioden tatsaechlich verschiedene Werte liefern (kein flacher Preisverlauf der identische ATR-Werte erzeugt).

## Offene Punkte

### 1. PromptBuilder Hardcoded Labels (ESKALATION)
`PromptBuilder.java` (Zeilen 414-420) hardcodiert:
- `"RSI(14,5m)"` -> `indicatorResult.rsi14_5m()`
- `"ATR(14,5m)"` -> `indicatorResult.atr14_5m()`
- `"ADX(14,5m)"` -> `indicatorResult.adx14_5m()`
- `"+DI(14,5m)"` -> `indicatorResult.plusDi14_5m()`
- `"-DI(14,5m)"` -> `indicatorResult.minusDi14_5m()`

Wenn Perioden auf z.B. 10 geaendert werden, erhaelt der LLM irrefuehrende Labels. Fix erfordert PromptBuilder-Refactoring (KpiProperties als Parameter). **Eigene Story empfohlen**, nicht in ODIN-079 Scope.

### 2. Morning-ATR Anchor Shift
Kuerzere ATR-Periode = frueherer Warmup = morningAtr wird frueher im Handelstag erfasst. Dies aendert die Semantik von `atrDecayRatio` leicht. Kein Bug, aber bei Perioden-Wechsel zu beachten.

### 3. Schwellen-Rekalibrierung bei Perioden-Aenderung
Schwellen wie `extensionRsiThreshold=78` sind auf RSI(14) kalibriert. RSI(9) ist volatiler -> haeufigere Threshold-Verletzungen. Perioden-Aenderung erfordert parallele Schwellen-Anpassung. Dies ist im Backtest-Rahmen zu behandeln, nicht hier.

### 4. DB-Spaltennamen (indicator_snapshot)
Spalten heissen `rsi14`, `atr14`, `adx14`. Bei verschiedenen Backtest-Konfigurationen ist die verwendete Periode nicht aus der DB ablesbar. Langfristig: Periode in Run-Konfiguration speichern.

## ChatGPT-Sparring

### Gefragt: Edge Cases fuer kurze Perioden, fehlende Testszenarien

### Vorgeschlagen und Bewertet:

| Vorschlag | Beurteilung | Umgesetzt? |
|-----------|-------------|------------|
| Config validation: period <= 0 rejected | Bereits durch @Min(1) geloest | Nein (existiert) |
| period=1 pathologisch | Out of scope, @Min(1) erlaubt es, kein Trading-Risiko da Defaults bei 14 bleiben | Nein |
| Period > maxBarCount -> permanent NaN | Kein Bug, da warmupComplete=false -> kein Trading moeglich. Bestehendes Verhalten korrekt | Nein |
| Flat-price series -> assertNotEquals flaky | Gut erkannt. Testdaten verwenden Sinus-Modulation, daher kein Risiko | Nein (testdata design) |
| PromptBuilder hardcoded "ATR(14)" labels | Sehr guter Fund! Als offener Punkt dokumentiert | Nein (eigene Story) |
| Multiple engines sharing state | Engines sind per-Pipeline, kein Shared State. Kein Risiko | Nein |
| RSI warmup suppression (ta4j returns 0 unstable bars) | KpiEngine hat eigene Warmup-Logik (period+1 minimum bars). Korrekt | Nein |
| ATR-derived threshold floors and NaN safety | Bereits in ExitRulesTest, EntryRulesTest getestet | Nein (existiert) |

## Gemini-Review

### Dimension 1: Code-Review
**Ergebnis: Keine Bugs gefunden.**
- Perioden-Propagation korrekt
- DI-Indikatoren korrekt an adxPeriod gebunden
- Exception-Handling und NaN-Fallback robust
- Warmup-Constraints mathematisch korrekt (period+1 fuer ATR/RSI, 2*period fuer ADX/DI)

### Dimension 2: Konzepttreue
**Ergebnis: Vollstaendig konform.**
- @Min(1) Validierung vorhanden
- Properties->Engine-Propagation verifiziert
- Backward-Compatibility durch Beibehaltung der Feldnamen
- JavaDoc-Updates korrekt

### Dimension 3: Praxis-Review
Findings (alle als offene Punkte dokumentiert):
1. **Morning-ATR Anchor Shift**: Kuerzere Periode -> frueherer Warmup -> anderer Baseline-Zeitpunkt
2. **Schwellen-Invalidierung**: RSI/ADX-abhaengige Schwellen sind auf Periode 14 kalibriert
3. **DB-Ambiguitaet**: Spalten mit "14" im Namen, tatsaechliche Periode nicht ablesbar
4. **LLM-Prompt-Labels**: Bekannter Fund, bereits dokumentiert

Alle Findings sind Konsequenzen einer Perioden-Aenderung, keine Implementation-Fehler. Perioden bleiben auf 14 bis Backtest-Ergebnisse vorliegen.

## Backtest-Vergleichsmethodik

### Ziel
Systematischer Vergleich von ATR(14) vs. ATR(10) vs. ATR(7), RSI(14) vs. RSI(9), ADX(14) vs. ADX(10) auf historischen Intraday-Daten.

### Parameter-Kombinationen
| Kombination | ATR | RSI | ADX | Beschreibung |
|-------------|-----|-----|-----|-------------|
| Baseline | 14 | 14 | 14 | Aktuelle Default-Konfiguration |
| ATR-responsiv | 10 | 14 | 14 | Nur ATR responsiver |
| ATR-aggressiv | 7 | 14 | 14 | ATR sehr responsiv |
| RSI-responsiv | 14 | 9 | 14 | Nur RSI responsiver |
| ADX-responsiv | 14 | 14 | 10 | Nur ADX responsiver |
| Balanced | 10 | 9 | 10 | Alle Perioden verkuerzt |
| Aggressive | 7 | 9 | 10 | Maximale Responsivitaet |

### Metriken pro Kombination
1. **Sharpe Ratio (Daily)**: Risikoadjustierte Rendite
2. **Win-Rate**: Anteil profitabler Trades
3. **Max Drawdown**: Groesster Peak-to-Trough
4. **Average Slippage**: Durchschnittliche Abweichung Entry/Exit vs. Signal
5. **Trade Count**: Anzahl der generierten Trades (mehr ist nicht automatisch besser)
6. **Average R-Multiple**: Durchschnittliches Risiko-Vielfaches pro Trade
7. **Profit Factor**: Brutto-Gewinne / Brutto-Verluste

### Vergleichsmethodik
1. **Gleiche Datenbasis**: Mindestens 20 Handelstage mit verschiedenen Marktphasen
2. **Gleiche Instrumente**: Mindestens 5 High-Beta US-Aktien (z.B. IREN, TSLA, NVDA, AMD, META)
3. **Neighbor-Test (+-10%)**: Fuer jede vielversprechende Konfiguration auch benachbarte Werte testen (z.B. ATR=10 -> auch ATR=9 und ATR=11). Wenn Ergebnis stark variiert, ist der Wert nicht robust
4. **Walk-Forward-Validierung**: Training auf 70% der Daten, Test auf 30% (nicht umgekehrt!)
5. **Anti-Overfitting-Check**: Ergebnis muss auf Out-of-Sample-Daten aehnlich sein wie auf In-Sample

### Schwellen die bei Perioden-Aenderung rekalibriert werden muessen
- `extensionRsiThreshold` (default 78): RSI(9) ist volatiler
- `adxTrendThreshold` (default 20): ADX(10) ist volatiler
- `adxMatureThreshold` (default 25): Gleiche Ueberlegung
- `adxLateThreshold` (default 40): Gleiche Ueberlegung
- ATR-basierte Schwellen (stopLoss, trailingStop, VWAP-Proximity): Sind als ATR-Vielfache definiert -> Verhaeltnis bleibt aehnlich, aber absolute Werte aendern sich

### Entscheidungskriterium
Eine kuerzere Periode ist NUR dann besser, wenn sie ueber verschiedene Instrumente und Marktphasen KONSISTENT bessere Metriken liefert. Ein einzelner guter Tag auf einem einzelnen Instrument reicht NICHT (Overfitting-Risiko).
