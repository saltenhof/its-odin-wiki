# Protokoll: ODIN-096 — Bar-Resolution-Parity

## Working State
- [x] Initiale Implementierung (4 Module: odin-data, odin-brain, odin-core, odin-backtest)
- [x] Unit-Tests geschrieben (13 neue Tests, 60 gesamt)
- [x] ChatGPT-Sparring fuer Edge Cases und Testluecken
- [x] Gemini-Review Dimension 1 (Code Quality / MOSES R1-R13)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis / Trading-System-Perspektive)
- [x] Review-Findings eingearbeitet (Config-Validierung, Backtest fail-fast)
- [x] Commit & Push

## Design-Entscheidungen

### Unified Aggregation Path
- BAR_APPROXIMATED-Bypass in DataPipelineService komplett entfernt. Alle Bars (Live-Ticks und Backtest-Replay) durchlaufen jetzt `barAggregator.onOneMinBar()`.
- BAR_APPROXIMATED bleibt als semantischer Marker erhalten (OFI-Skip, kein Tick-Level-Daten), steuert aber NICHT den Datenpfad.
- Resultat: Live und Backtest nutzen identischen Code fuer Aggregation, KPI-Berechnung und Signalgenerierung.

### Decision-Bar Konfigurierbarkeit
- `odin.data.decision-bar-timeframe-s` steuert den Decision-Bar-Timeframe (180=3m oder 300=5m).
- Wert wird per Konstruktor durch die POJO-Kette propagiert: DataPipelineService -> RollingDataBuffer, PipelineFactory -> KpiEngine -> BarSeriesManager.
- Validierung via `@AssertTrue` in DataProperties: nur 180 oder 300 erlaubt. Verhindert stille Fehlkonfiguration (kein Snapshot-Trigger bei ungueltigem Wert).

### Backtest-Input auf 1m beschraenkt
- `BacktestRunner.runWithOverrides()` wirft `IllegalArgumentException` bei barInterval != ONE_MINUTE.
- Begruendung: Hoehere Timeframes (3m, 5m, 15m, 30m) werden durch BarAggregator-Kaskade erzeugt. Direktes Laden von 5m-Bars wuerde falsche Open/Close-Zeiten und kaputte Aggregation produzieren.
- `application.properties` Default: `odin.backtest.bar-interval=ONE_MINUTE`.

### BarInterval Enum Erweiterung
- THREE_MINUTES(180) hinzugefuegt fuer zukuenftige Nutzung (z.B. explizite 3m-Daten-Downloads).
- Aktuell nur ONE_MINUTE fuer Backtest-Input zugelassen.

## Geaenderte Dateien

### odin-data
- `DataPipelineService.java`: BAR_APPROXIMATED-Bypass entfernt, unified path durch BarAggregator, decisionBarProduced-Logik fuer 3m/5m
- `RollingDataBuffer.java`: Konstruktor mit `decisionBarTimeframeS`, `getDecisionBars()` dynamisch (180->3m, 300->5m)
- `DataProperties.java`: `@AssertTrue isValidDecisionBarTimeframe()` — nur 180 oder 300 erlaubt

### odin-brain
- `BarSeriesManager.java`: Konstruktor mit `decisionBarDurationSeconds`, `resolveDuration()` konfigurierbar, `DURATION_3M_SECONDS` public
- `KpiEngine.java`: Neuer Konstruktor mit `decisionBarDurationSeconds`, Weitergabe an BarSeriesManager

### odin-core
- `PipelineFactory.java`: `new KpiEngine(brainProperties, diagnosticSink, dataProperties.decisionBarTimeframeS())`

### odin-backtest
- `BacktestRunner.java`: Fail-fast bei barInterval != ONE_MINUTE, fail-fast bei fehlenden 1m-Bars
- `BarInterval.java`: THREE_MINUTES(180) hinzugefuegt
- `application.properties`: bar-interval Default auf ONE_MINUTE geaendert

### Tests
- `RollingDataBufferTest.java`: +6 Tests (3m/5m Decision-Bar Konfiguration, Fallback, getThreeMinBarCount)
- `BarSeriesManagerTest.java`: +4 Tests (3m/5m Duration-Konfiguration, 1m/5m bleiben fix)
- `KpiEngineTest.java`: +3 Tests (5m Decision Duration, Default 3m Duration, Bar-Series-Duration-Verifikation)

## ChatGPT-Sparring

### Vorgeschlagene Edge Cases (7 Kategorien)
ChatGPT hat umfangreiche Edge Cases identifiziert. Bewertung:

**Eingearbeitet:**
1. Inkonsistenter Fallback bei unbekanntem Timeframe-Config -> Validierung in DataProperties (`@AssertTrue`, nur 180/300)
2. Backtest mit 3m/5m Input-Bars fehlinterpretiert -> Fail-fast in BacktestRunner (nur ONE_MINUTE erlaubt)

**Akzeptiert (kein Handlungsbedarf):**
3. Decision=5m Doppelung (decisionBars == fiveMinBars im Snapshot): Bewusste Semantik. KpiEngine und RulesEngine handhaben dies korrekt, da separate ta4j-Serien gepflegt werden. Keine Feature-Doppelzaehlung.
4. Warmup bleibt 5m-getrieben auch bei decision=3m: Korrekt. KPI-Warmup (RSI, ATR, ADX, Bollinger) benoetigt 5m-Bars. Decision-EMA-Warmup ist schneller, aber Trading darf erst starten wenn ALLE Indikatoren valid sind.
5. Shutdown flush ohne finalen Decision-Snapshot: Design-Entscheidung. EOD-Close wird durch CloseCountdown-Phase gesteuert, nicht durch letzte Decision-Bar.
6. TIMEFRAME_3M Konstante ohne zugehoerige Serie: Nur als Doku-Konstante; BarSeriesManager nutzt "decision" als logischen Alias.
7. Config-Wechsel zur Laufzeit: DataProperties ist immutabel (Record + Spring ConfigurationProperties). Kein Laufzeit-Wechsel moeglich.

## Gemini-Review

### Dimension 1: Code Quality (MOSES R1-R13)
**Status: Conditional Pass**

**Findings:**
1. (LOW) Timeframe als String statt Enum in BarSeriesManager -> AKZEPTIERT: Bestandscode, Refactoring waere eigene Story. Aktuell durch Konstanten und switch-Expressions abgesichert.
2. (LOW) Magic Numbers in BacktestRunner.computeSeasonalityCurve (1.5, 5) -> AKZEPTIERT: Bestandscode ausserhalb ODIN-096 Scope. Notiert fuer spaetere Bereinigung.

**Positive Bewertung:** Kein `var`, Records durchgaengig, Switch-Expressions, `.toList()`, JavaDoc vollstaendig.

### Dimension 2: Konzepttreue
**Status: Pass (Volle Konzepttreue)**

Alle Konzept-Vorgaben erfuellt:
- H1 (RollingDataBuffer): Erfuellt
- H2 (BarSeriesManager): Erfuellt
- H3 (SimulationRunner): Erfuellt (via BacktestRunner fail-fast)
- H4 (BacktestProperties): Erfuellt
- BAR_APPROXIMATED Bypass: Entfernt
- Fail-fast: Implementiert
- Config-Namespace: Korrekt
- Validierung: Implementiert

### Dimension 3: Praxis (Trading-System-Perspektive)
**Status: Pass**

**Bewertung:**
1. Trading-Relevanz: Sehr hoch positiv. Sim-to-Live Divergence eliminiert.
2. P&L-Impact: Wahrscheinlich realistischere (moeglicherweise niedrigere) Backtest-P&L durch korrekte Intra-Bar-Stopps und Event-Detection. Dies ist eine Verbesserung, keine Verschlechterung.
3. Risiken: Erhoehte Sensitivitaet gegenueber 1m-Datenqualitaet. Bestehende DQ-Gates decken dies ab.
4. Latenzen: Moderate Verlangsamung (5x mehr Bar-Events) — vernachlaessigbar bei 390 RTH-Minuten/Tag.
5. Empfehlungen: Intra-Bar Stop-Loss Optimierung, Daten-Scrubbing fuer historische 1m-Bars.

## Test-Ergebnisse
- RollingDataBufferTest: 19/19 PASSED (13 bestehend + 6 neu)
- BarSeriesManagerTest: 18/18 PASSED (14 bestehend + 4 neu)
- KpiEngineTest: 23/23 PASSED (20 bestehend + 3 neu)
- Gesamt: 60 Tests, 0 Failures, 0 Errors
- Build: mvn clean install -DskipTests — BUILD SUCCESS (alle 11 Module)
