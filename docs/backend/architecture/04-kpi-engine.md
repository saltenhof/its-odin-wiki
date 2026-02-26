# ODIN — Kapitel 4: KPI-Engine & Quantitative Validierung

Version: 0.3 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R2 — P0/P1 Fixes (Warmup-Flow, Volume-Ratio, Intent-Garantie, R/R-Kontrakt, EventLog-Failure, Sim-Determinismus, Delta-Regeln)

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt die KPI-Engine und die quantitative Validierung (Quant Validation) in odin-brain. Die KPI-Engine berechnet technische Indikatoren aus dem MarketSnapshot und stellt sie fuer die nachgelagerten Entscheidungsschichten (Rules Engine, Decision Arbiter) bereit. Die Quant Validation bewertet Trade-Intents anhand eines gewichteten Scoring-Modells mit Veto-Mechanismus.

**Modulzuordnung:** `odin-brain` (Package: `de.odin.brain.kpi`, `de.odin.brain.quant`)

**Zentrale Abhaengigkeiten:**
- `MarketSnapshot` (von odin-data via `MarketSnapshotEvent`) — Rohdaten und Derived Features (inkl. VWAP, SpreadStats)
- `MarketClock` (Port aus odin-api) — Zeitreferenz fuer alle zeitbasierten Berechnungen
- `EventLog` (Port aus odin-api) — Persistierung von IndicatorResult und QuantScore
- `RunContext` (aus odin-api) — runId als Join-Key

---

## 2. Stellung im Decision-Cycle

Die KPI-Engine ist der **erste Schritt** im Decision-Cycle nach dem MarketSnapshotEvent. Der vollstaendige Ablauf pro Pipeline bei jedem Decision-Bar-Close (3m oder 5m):

```
odin-data: MarketSnapshotEvent publiziert
       │
       v
1. KPI-Engine (odin-brain)              ← DIESES KAPITEL
   ├── ta4j-BarSeries aktualisieren
   ├── Indikatoren berechnen
   └── IndicatorResult erzeugen
       │
       v
2. Rules Engine (odin-brain, Kap 6)
   ├── Pattern-State-Machines
   └── Entry/Exit-Intent erzeugen
       │
       v
3. Quant Validation (odin-brain)         ← DIESES KAPITEL
   ├── quant_score berechnen
   └── Veto-Pruefung
       │
       v
4. Decision Arbiter (odin-brain)
   └── Finaler TradeIntent
       │
       v
5. Risk Gate + OMS (odin-execution, Kap 7)
```

### Trigger und Barrier-Kopplung

Die KPI-Engine wird **synchron** innerhalb des Decision-Cycle getriggert — nicht asynchron oder event-basiert. Der `MarketSnapshotEvent` ist der Trigger. Die KPI-Berechnung muss abgeschlossen sein, bevor die Rules Engine startet.

**In der Simulation:** Der SimulationRunner (Kap 0, Bar-Close-Barrier) gibt das naechste Event erst frei, wenn der gesamte Decision-Cycle (inkl. KPI → Rules → Quant → Arbiter → Risk → OMS) abgeschlossen ist. Die KPI-Engine ist der erste synchrone Schritt in dieser Kette.

**Im Live-Betrieb:** Der Decision Loop ist single-threaded pro Pipeline (Kap 0, Abschnitt 6). Die KPI-Berechnung blockiert den Pipeline-Thread bis zum Abschluss — bei 2 Timeframes und < 1ms pro Indikator ist das unkritisch.

### Warmup-Verhalten im Decision-Cycle

Der Decision-Cycle laeuft auf **jedem** MarketSnapshotEvent — auch waehrend der Warmup-Phase. Die KPI-Engine erzeugt immer ein `IndicatorResult` (mit `warmupComplete = false` waehrend Warmup). Die nachgelagerten Schritte reagieren per Guard:

| Schritt | Verhalten waehrend Warmup |
|---------|--------------------------|
| KPI-Engine | Laeuft normal. IndicatorResult mit NaN-Werten und `warmupComplete = false` |
| Rules Engine | Prueft `warmupComplete`. Bei `false`: kein TradeIntent erzeugt → Cycle endet hier |
| Quant Validation | Wird nicht aufgerufen (kein Intent vorhanden) |
| Decision Arbiter | Wird nicht aufgerufen (kein Intent vorhanden) |

> **Kein separater Warmup-Modus:** Der Decision-Cycle ist immer derselbe Codepfad. Warmup wirkt als Guard in der Rules Engine, nicht als separater Pipeline-State. Das IndicatorResult waehrend Warmup dient dem Monitoring (Frontend kann Indikator-Aufbau visualisieren).

---

## 3. ta4j-Wrapper

### Designentscheidung (A3: ENTSCHIEDEN)

ODIN nutzt ta4j ueber einen **leichtgewichtigen Wrapper** (keine direkte ta4j-Abhaengigkeit in Fachklassen ausserhalb von odin-brain). Der Wrapper bietet:

- **Einheitliche API:** Indikatoren werden ueber ein ODIN-eigenes Interface (`KpiEngine`) abgefragt, nicht ueber ta4j-Klassen
- **Testbarkeit:** `KpiEngine` kann in Unit-Tests gemockt werden, ohne ta4j auf dem Classpath
- **BarSeries-Management:** Synchronisation zwischen `MarketSnapshot`-Bars und ta4j-`BarSeries`

### Abgrenzung zu odin-data

Die KPI-Engine berechnet **interpretative Indikatoren** (EMA, RSI, ATR, Bollinger, ADX) — Werte die Interpretation und technische Analyse darstellen. **Rohe Derived Features** (VWAP, SpreadStats, Intraday-Extrema) werden in odin-data berechnet und ueber den MarketSnapshot bereitgestellt (Kap 2, Abschnitt 8).

| Kategorie | Berechnet in | Beispiele |
|-----------|-------------|-----------|
| Rohe Derived Features | odin-data (Buffer/Akkumulator) | VWAP, SpreadStats, Intraday-High/Low, cumulativeVolume |
| Technische Indikatoren | odin-brain (KPI-Engine, ta4j) | EMA, RSI, ATR, Bollinger, ADX |
| Scoring/Bewertung | odin-brain (Quant Validation) | quant_score, Veto-Flags |

---

## 4. BarSeries-Management und Zeitregeln

### BarSeries pro Pipeline und Timeframe

Pro Pipeline und Timeframe wird eine eigene ta4j-`BarSeries` gefuehrt. Die Bars kommen aus dem `MarketSnapshot` (Kap 2, Abschnitt 7):

| Timeframe | Snapshot-Feld | BarSeries-Kapazitaet | Indikatoren |
|-----------|--------------|---------------------|-------------|
| Decision-Bar (3m oder 5m) | `decisionBars` | 130 Bars bei 3m / 78 Bars bei 5m (1 Handelstag) | EMA(9), EMA(21) |
| 5-Min (Fixer KPI-Timeframe) | `fiveMinBars` | 78 Bars (1 Handelstag) | RSI(14), ATR(14), Bollinger(20,2), ADX(14) |

> **Drei-Layer-Trennung:** EMA(9) und EMA(21) werden auf Decision-Bars berechnet (3m oder 5m, je nach Konfiguration). ATR(14), ADX(14), RSI(14) und Bollinger(20,2) werden **IMMER auf 5m-Bars** berechnet — unabhaengig davon, ob der Decision-Layer 3m oder 5m ist. Diese Trennung stellt sicher, dass die Standard-Risikoindikatoren konsistent auf einem fixen Timeframe basieren.

> **Kein 15-Min-Timeframe in v1:** Die v0.1 listete 15-Min-Indikatoren, aber der Mehrwert gegenueber 5-Min ist fuer v1 nicht belegt. Die BarSeries-Verwaltung unterstuetzt beliebige Timeframes — 15-Min kann spaeter ergaenzt werden, wenn Profiling/Backtesting den Bedarf zeigt.

Bei jedem neuen Decision-Bar-Close (= jeder `MarketSnapshotEvent`) werden die BarSeries wie folgt aktualisiert:

1. **Delta-Extraktion:** Neue Bars aus dem Snapshot extrahieren. Delta wird ueber `barEndTime` bestimmt — nur Bars mit `barEndTime > lastProcessedBarEndTime[timeframe]` werden eingefuegt. Bereits verarbeitete Bars werden uebersprungen (Duplikat-Schutz)
2. **Timeframe-spezifisches Update:** Decision-Bar-BarSeries wird bei jedem MarketSnapshotEvent aktualisiert (neuer Decision-Bar liegt immer vor). 5-Min-BarSeries wird **nur** aktualisiert wenn ein neuer 5-Min-Bar im Snapshot vorliegt (erkennbar an neuem `barEndTime` in `fiveMinBars`). Bei Decision-Layer = 3m liegt nicht bei jedem Decision-Cycle ein neuer 5m-Bar vor — die 5m-BarSeries bleibt zwischen 5m-Grenzen unveraendert
3. **Indikator-Berechnung:** ta4j berechnet alle angehangenen Indikatoren automatisch bei Zugriff (Lazy Evaluation)

### Zeitregeln

| Aspekt | Regelung |
|--------|---------|
| Zeitquelle | Alle Bar-Timestamps kommen aus dem `MarketSnapshot.marketTime` (MarketClock-basiert, Exchange-TZ). **Kein `Instant.now()`** |
| RTH-Filterung | Indikatoren nur auf RTH-Bars. Die Data Pipeline (Kap 2) liefert bereits RTH-gefilterte Bars im Snapshot — die KPI-Engine muss keine eigene RTH-Pruefung durchfuehren |
| Bar-Ende-Konvention | Close-Time = Beginn der naechsten Bar (z.B. 5-Min-Bar 09:30–09:35 hat Close-Time 09:35). Konsistent mit odin-data (Kap 2, Abschnitt 4) |
| Fehlende Bars | Kein Gap-Fill — BarSeries ueberspringt fehlende Bars. ta4j-Indikatoren tolerieren Luecken. Bei > 3 aufeinanderfolgenden fehlenden 1m-Bars wird `DataFlag.DATA_GAP` im IndicatorResult gesetzt (Optional: Entry-Block per Config `odin.kpi.block-entry-on-gap=true`). **Abgrenzung zu Market Halts:** Exchange-Halts und Trading Pauses werden von odin-data (StaleQuoteGate, Kap 2) erkannt und via `STALE_WARNING`/`ESCALATE` behandelt. `DATA_GAP` in der KPI-Engine erkennt nur fehlende Bars die durch DQ-Rejects oder Datenuebertragungsprobleme entstehen — nicht durch regulaere Handelspausen |
| Warmup | Indikatoren gelten erst als valid nach `period + 1` Bars. Vorher: `NaN`/unstable. `warmupComplete = false` → Rules Engine erzeugt keinen Intent (siehe Abschnitt 2, Warmup-Verhalten) |

### SOD-Reset (Start-of-Day)

Bei Tagesstart (Pipeline-Initialisierung) werden **alle ta4j-BarSeries geleert**. Die feingranularen Buffer (1m, 5m) fuer den aktuellen Tag werden ausschliesslich aus Live-/Replay-Daten aufgebaut (Kap 2, Abschnitt 6: keine Disaggregation). Die KPI-Engine beginnt jeden Tag mit leerem State und baut Indikator-Werte inkrementell auf.

---

## 5. Indikator-Toolkit

### Kern-Indikatoren

| Indikator | Timeframe | Verwendung | ta4j-Klasse |
|-----------|-----------|-----------|-------------|
| EMA(9) | Decision-Bar (3m oder 5m) | Kurzfristiger Trend | `EMAIndicator` |
| EMA(21) | Decision-Bar (3m oder 5m) | Kurzfristiger Trend (Kreuzung mit EMA9) | `EMAIndicator` |
| RSI(14) | 5-Min (fixer KPI-Timeframe) | Ueberkauft/Ueberverkauft-Check | `RSIIndicator` |
| ATR(14) | 5-Min (fixer KPI-Timeframe) | Volatilitaet, Stop-Distance, Position Sizing | `ATRIndicator` |
| Bollinger Bands(20,2) | 5-Min (fixer KPI-Timeframe) | Ueberhitzungserkennung (Runner-Trailing) | `BollingerBandsMiddleIndicator` etc. |
| ADX(14) | 5-Min (fixer KPI-Timeframe) | Trendstaerke-Messung | `ADXIndicator` |

### Derived Values (von KPI-Engine berechnet)

| Wert | Berechnung | Verwendung |
|------|-----------|-----------|
| ATR-Decay-Ratio | `current_5min_ATR / morning_5min_ATR` | LLM-Kontext, Taktik-Anpassung |
| Volumen-Ratio | `lastBarVolume / SMA(barVolume, 20)` | Anomalie-Detection, Flush/Coil-Erkennung |

> **ATR-Decay-Ratio Baseline:** `morning_5min_ATR` wird gesetzt auf den **ersten validen ATR(14)-Wert nach Warmup-Abschluss** (d.h. der ATR-Wert beim 15. 5-Min-Bar). Bis dahin ist `atrDecayRatio = NaN`. Die Baseline wird einmal pro Tag gesetzt und aendert sich nicht mehr.

> **Volumen-Ratio Referenzwert:** Der Nenner `SMA(barVolume, 20)` ist der gleitende Durchschnitt ueber die letzten 20 Decision-Bars (3m oder 5m). Das ergibt einen stabilen, bar-bezogenen Vergleichswert ohne zeitabhaengigen Drift (anders als cumulativeVolume, das ueber den Tag waechst).

### VWAP — Konsument, nicht Produzent

Die KPI-Engine **konsumiert** den VWAP aus dem `MarketSnapshot.vwap` — sie berechnet ihn **nicht** selbst. Der VWAP ist Source-of-Truth in odin-data (Kap 2, Abschnitt 6+8). Die KPI-Engine nutzt den VWAP als Input fuer:
- Trend-Alignment-Check in der Quant Validation (Preis vs. VWAP)
- Bereitstellung an die Rules Engine (Opportunity-Zone-Erkennung)

### SpreadStats — Konsument, nicht Produzent

Analog: `SpreadStats` (min/max/avg Spread) kommen aus dem `MarketSnapshot.spreadStats`. Die Quant Validation nutzt sie fuer den Spread-Veto-Check.

---

## 6. IndicatorResult (Output)

Bei jedem Decision-Cycle erzeugt die KPI-Engine ein `IndicatorResult`:

```
IndicatorResult (Record, immutable, in odin-api)
├── runId               : String (Join-Key zum RunContext)
├── instrumentId        : String
├── barTime             : Instant (Close-Time der Decision-Bar = Bar-End, konsistent mit Kap 2)
├── ema9                : double (NaN wenn noch nicht valid, berechnet auf Decision-Bars)
├── ema21               : double (NaN wenn noch nicht valid, berechnet auf Decision-Bars)
├── rsi14_5m            : double (NaN wenn noch nicht valid)
├── atr14_5m            : double (NaN wenn noch nicht valid)
├── bollingerUpper_5m   : double
├── bollingerMiddle_5m  : double
├── bollingerLower_5m   : double
├── adx14_5m            : double (NaN wenn noch nicht valid)
├── atrDecayRatio       : double (NaN wenn morning_ATR noch nicht gesetzt)
├── volumeRatio         : double
├── vwap                : double (durchgereicht aus Snapshot)
├── warmupComplete      : boolean (alle Indikatoren valid?)
└── flags               : Set<DataFlag> (durchgereicht aus Snapshot + eigene: BAR_APPROXIMATED, DATA_GAP)
```

> **barTime:** Die Close-Time der ausloesenden Decision-Bar (= Bar-End = Beginn der naechsten Bar, z.B. 09:35 fuer die 09:34–09:35 Bar). Identisch mit `MarketSnapshot.marketTime`. Dieser Wert ist der primaere Join-Key ueber alle Module (EventLog, OMS, Broker-Reports).

> **warmupComplete:** Wird `true` sobald alle Indikatoren gueltige Werte haben (d.h. genuegend Bars fuer alle Perioden vorhanden). Bei `false` erzeugt die Rules Engine keinen TradeIntent (Guard, siehe Abschnitt 2).

> **flags:** Durchgereichte DataFlags aus dem MarketSnapshot (z.B. `BAR_APPROXIMATED`) plus eigene Flags der KPI-Engine (z.B. `DATA_GAP` bei > 3 fehlenden Bars). Nachgelagerte Konsumenten (Rules, Quant, Arbiter) koennen darauf reagieren.

---

## 7. Quantitative Validierung (Quant Validation)

Die Quant Validation laeuft **nach** der Rules Engine und **vor** dem Decision Arbiter (siehe Abschnitt 2). Sie bewertet die Qualitaet eines Trade-Intents, den die Rules Engine erzeugt hat.

> **v1 Scope: Long-Only.** Alle Scoring-Funktionen und Veto-Regeln sind fuer Long-Entries definiert (Kap 0: Long-Only System). Short-spezifische Spiegelregeln (RSI < 25, Preis < VWAP - X%, EMA9 > EMA21 als Short-Veto) werden in v2 ergaenzt falls Short-Selling eingefuehrt wird.

### Input-Kontrakt

Die Quant Validation erhaelt einen `TradeIntent` von der Rules Engine. Dieser muss folgende Felder enthalten:

| Feld | Typ | Herkunft | Beschreibung |
|------|-----|----------|-------------|
| `entryPrice` | double | Rules Engine (currentPrice aus Snapshot) | Beabsichtigter Einstiegspreis |
| `stopPrice` | double | Rules Engine (1.5 × ATR(14) unter Entry) | Initial-Stop-Loss |
| `targetPrice` | double | Rules Engine (aus LLM target_price oder synthetisch 2 × ATR(14) ueber Entry) | Kursziel |
| `side` | TradeDirection | Rules Engine | Immer `LONG` in v1 |

> **R/R-Berechnung:** `riskRewardRatio = (targetPrice - entryPrice) / (entryPrice - stopPrice)`. Alle Preise sind **vor** Slippage/Fees — die Quant Validation bewertet das theoretische Setup. Position Sizing und Kostenrechnung sind Aufgabe des Risk Gate (Kap 7).

### Ein-Intent-pro-Bar Garantie

Pro Decision-Cycle (= pro MarketSnapshotEvent = pro Decision-Bar-Close) erzeugt die Rules Engine **maximal einen TradeIntent** pro Pipeline. Damit ist das Tupel `(runId, instrumentId, barTime)` ein eindeutiger Key fuer den QuantScore.

**Tie-Break bei konkurrierenden Signalen:** Falls in einem Decision-Cycle sowohl ein Exit-Signal (z.B. Stop-Hit, EOD-Flat) als auch ein Entry-Signal vorliegen, gilt die Prioritaet: **Exit vor Entry**. Es wird kein neuer Entry eroeffnet solange eine bestehende Position geschlossen werden muss. Falls mehrere Entry-Patterns gleichzeitig feuern (v1: unwahrscheinlich bei Long-Only mit einem Setup-Typ pro Instrument), entscheidet die Rules Engine nach konfigurierter Pattern-Prioritaet (Details in Kap 6). Die Quant Validation sieht immer nur den einen resultierenden Intent.

### Scoring-Modell

Der `quant_score` (0.0–1.0) bewertet die Qualitaet eines Trade-Intents:

| Check | Gewichtung | Score-Logik (Long) |
|-------|-----------|-------------|
| Trend-Alignment (EMA, VWAP) | 0.25 | 1.0 wenn EMA(9) > EMA(21) AND Preis > VWAP. 0.5 wenn nur eines zutrifft. 0.0 wenn keines |
| Overbought (RSI) | 0.20 | 1.0 bei RSI ∈ [30, 65]. Linear fallend: `score = 1.0 - (RSI - 65) / 15` fuer RSI ∈ (65, 80]. 0.0 bei RSI > 80 |
| Volumen-Bestaetigung | 0.20 | 1.0 bei Volume-Ratio >= 1.5. Linear fallend: `score = (volRatio - 0.5) / 1.0` fuer volRatio ∈ [0.5, 1.5). 0.0 bei volRatio < 0.5 |
| Spread-Pruefung | 0.15 | 1.0 bei Spread <= 0.1%. Linear fallend: `score = 1.0 - (spread - 0.1) / 0.4` fuer Spread ∈ (0.1%, 0.5%]. 0.0 bei Spread > 0.5% |
| Risk/Reward-Ratio | 0.20 | 1.0 bei R/R >= 2.0. Linear fallend: `score = (rr - 1.5) / 0.5` fuer R/R ∈ [1.5, 2.0). 0.0 bei R/R < 1.5 |

> **Clamping:** Alle Scores werden auf [0.0, 1.0] geclampt.

> **NaN-Policy (komponentenspezifisch):**
> - **Pflicht-Komponenten** (Trend-Alignment, RSI, Volume, R/R): Wenn eine dieser Komponenten NaN liefert (z.B. Warmup nicht abgeschlossen), wird der gesamte `quant_score = NaN` → Intent blockiert
> - **Optionale Komponenten** (Spread bei `BAR_APPROXIMATED`): NaN blockiert **nicht**. Die Komponente wird aus der Gewichtung entfernt und die Gewichte der verbleibenden Pflicht-Komponenten proportional renormalisiert (z.B. ohne Spread: Trend 0.294, RSI 0.235, Volume 0.235, R/R 0.235)
> - **Zusammenspiel:** Im Bar-Replay-Modus ist Spread = NaN → Spread-Check deaktiviert, Gewichte renormalisiert. Alle anderen Indikatoren (EMA, RSI, ATR, ADX) sind exakt (bar-basiert) und liefern kein NaN nach Warmup

**Schwelle:** `quant_score >= 0.50` → Trade erlaubt. `quant_score < 0.50` → Trade blockiert.

### Hard-Veto-System

Einzelne Checks haben ein **absolutes Veto-Recht** unabhaengig vom Gesamtscore:

| Veto-Bedingung (Long) | Default | Begruendung |
|----------------------|---------|-------------|
| Spread > X% | 0.5% | Liquiditaet zu gering, Slippage-Risiko |
| RSI > X | 75 | Ueberkauft, Reversal-Risiko |
| EMA(9) < EMA(21) auf Decision-Bars | — | Kurzfristiger Abwaertstrend |
| Preis > VWAP + X% | 2% | Zu weit vom Fair Value entfernt |

> **Alle Schwellenwerte sind konfigurierbar** (Properties). Die Defaults sind konservativ gewaehlt. Fuer v1 gelten einheitliche Schwellenwerte fuer alle Instrumente.

> **Veto ist nicht verhandelbar:** Ein Hard-Veto blockiert den Trade unabhaengig davon, wie hoch der Gesamtscore ist.

### Spread-Check im Bar-Replay-Modus

Im Bar-Replay-Modus (Simulation) sind `SpreadStats` im Snapshot nicht verfuegbar (`BAR_APPROXIMATED`-Flag, Kap 2). Die Quant Validation handhabt dies wie folgt:

| Modus | Spread-Verfuegbarkeit | Verhalten |
|-------|----------------------|-----------|
| Live / Tick-Replay | Exakt (aus L2 + Ticks) | Normaler Spread-Check + Veto |
| Bar-Replay | Nicht verfuegbar (0.0) | **Spread-Check wird uebersprungen** (kein Veto, kein Score-Beitrag). Die Gewichtung der anderen Checks wird proportional hochskaliert |

> **Architektonische Entscheidung:** Keinen Spread erfinden. Im Bar-Replay fehlt die L2-/Tick-Information — ein geschaetzter Spread waere falsch und wuerde zu falschen Vetos oder falschem Vertrauen fuehren. Stattdessen wird der Check sauber deaktiviert und die Strategie muss im Tick-Replay-Modus validiert werden, wenn Spread-Sensitivitaet kritisch ist.

---

## 8. Interaction mit Regime-Confidence

Die Quant-Schwelle wird dynamisch an die LLM-Regime-Confidence gekoppelt:

| Regime-Confidence | Quant-Score-Schwelle | Begruendung |
|-------------------|---------------------|-------------|
| < 0.5 | Entry verboten (unabhaengig vom Score) | Regime unsicher |
| 0.5–0.7 | >= 0.70 | Hohe Quant-Huerde kompensiert schwache LLM-Confidence |
| > 0.7 | >= 0.50 | Standard-Huerde, da LLM sicher ist |

---

## 9. QuantScore (Output)

```
QuantScore (Record, immutable, in odin-api)
├── runId               : String (Join-Key zum RunContext)
├── instrumentId        : String
├── barTime             : Instant (MarketClock, Exchange-TZ)
├── overallScore        : double (0.0–1.0)
├── trendAlignmentScore : double
├── rsiScore            : double
├── volumeScore         : double
├── spreadScore         : double (NaN wenn BAR_APPROXIMATED)
├── rrRatioScore        : double
├── vetoActive          : boolean
├── vetoReasons         : List<String> (leer wenn kein Veto)
├── regimeConfidence    : double (durchgereicht vom LLM-Output)
├── effectiveThreshold  : double (dynamisch angepasst an Regime-Confidence)
├── approved            : boolean (overallScore >= effectiveThreshold AND !vetoActive)
└── flags               : Set<DataFlag> (durchgereicht, inkl. BAR_APPROXIMATED)
```

---

## 10. Simulation-Semantik

Die KPI-Engine verhaelt sich in Live und Simulation **identisch** (gleicher Codepfad, Kap 0 Abschnitt 3.1). Unterschiede ergeben sich ausschliesslich aus der Qualitaet der Eingabedaten im MarketSnapshot:

### Auswirkung von BAR_APPROXIMATED auf KPI

| Indikator/Feature | Tick-Replay | Bar-Replay | Kommentar |
|-------------------|-------------|------------|-----------|
| EMA(9), EMA(21) | Exakt (Decision-Bars aus Ticks gebaut) | Exakt (Decision-Bars direkt aus Feed) | EMA basiert auf Close-Preisen der Decision-Bars — identisch in beiden Modi |
| RSI(14) | Exakt | Exakt | Basiert auf Close-Preisen |
| ATR(14) | Exakt | Exakt | Basiert auf High/Low/Close — in Bars verfuegbar |
| Bollinger(20,2) | Exakt | Exakt | Basiert auf Close-Preisen |
| ADX(14) | Exakt | Exakt | Basiert auf High/Low/Close |
| ATR-Decay-Ratio | Exakt | Exakt | Verhaeltnis zweier ATR-Werte |
| Volumen-Ratio | Exakt | Exakt (bar-basiert) | lastBarVolume / SMA(barVolume, 20) — basiert auf Bar-Volumes, identisch in beiden Modi |
| VWAP (aus Snapshot) | Exakt | **Approximiert** (BAR_APPROXIMATED) | Aus odin-data, typicalPrice-Approximation |
| SpreadStats (aus Snapshot) | Exakt | **Nicht verfuegbar** (BAR_APPROXIMATED) | Kein L2/Tick-Daten |

> **Fazit:** Die meisten ta4j-Indikatoren (EMA, RSI, ATR, Bollinger, ADX) sind in beiden Modi exakt, da sie auf Bar-OHLCV basieren. Einschraenkungen betreffen primaer die Quant Validation (Spread-Check, VWAP-Deviation-Check).

### Determinismus-Garantie

Gegeben denselben MarketSnapshot (gleicher Input) produziert die KPI-Engine **deterministisch** dasselbe IndicatorResult. ta4j-Indikatoren sind zustandsbehaftet (sie akkumulieren ueber die BarSeries), aber der Zustand ergibt sich vollstaendig aus der Reihenfolge der Bars — und die ist durch die Bar-Close-Barrier determiniert.

### Determinismus der Regime-Confidence (Quant Validation)

Der QuantScore haengt ueber die dynamische Schwelle (Abschnitt 8) von `regimeConfidence` ab, die vom `LlmAnalyst`-Port kommt. Damit der QuantScore deterministisch ist, muss auch `regimeConfidence` deterministisch sein:

| Sim-Profil | Determinismus | Mechanismus |
|------------|---------------|-------------|
| **CachedAnalyst (Default)** | **Deterministisch** | Response aus Cache (Key: Hash aus Snapshot + Prompt-Version + Model-ID, Kap 0 Abschnitt 3.3). Gleicher Input → gleiche regimeConfidence |
| **Live-LLM** | **Nicht deterministisch** | Echte API-Calls, LLM-Output kann variieren (Temperature > 0). Fuer qualitative Analyse, nicht fuer reproduzierbare Backtests |

> **Fuer reproduzierbare Backtests ist CachedAnalyst Pflicht.** Im Live-LLM-Sim-Profil akzeptiert der Anwender bewusst Non-Determinismus (z.B. fuer A/B-Tests neuer Prompts).

---

## 11. EventLog-Integration

Die KPI-Engine schreibt folgende Events ins EventLog:

| Event | Volume | Drop-Policy | Begruendung |
|-------|--------|-------------|-------------|
| `IndicatorResult` | Niedrig (1 pro Bar-Close pro Pipeline) | **Best-effort** — darf bei Ueberlast gedroppt werden | Replay kann Indikatoren aus den Bars rekonstruieren (ta4j ist deterministisch) |
| `QuantScore` | Niedrig (1 pro Trade-Intent-Bewertung) | **Non-droppable** — Pflicht fuer Replay | QuantScore enthaelt Veto-Entscheidung, die den Trade blockiert oder erlaubt. Ohne QuantScore ist der Decision-Pfad nicht nachvollziehbar |

**Join-Keys und Idempotenz:** Alle Events tragen `(runId, instrumentId, barTime)` als eindeutigen Key. Durch die Ein-Intent-pro-Bar Garantie (Abschnitt 7) ist dieser Key fuer QuantScore ausreichend — kein zusaetzlicher `intentId` noetig. Bei Replay wird Idempotenz ueber den Key sichergestellt (Duplikate werden per Upsert-Semantik verworfen).

### EventLog-Failure-Semantik

Der EventLog-Schreibpfad folgt dem in Kap 1 (Abschnitt 11) und Kap 2 (Abschnitt 10) definierten Modell:

| Modus | Schreibverhalten | Failure-Reaktion |
|-------|-----------------|-----------------|
| **Simulation** | **Synchron** (innerhalb Bar-Close-Barrier). QuantScore wird geschrieben bevor der naechste Bar-Close freigegeben wird | EventLog-Failure = harter Fehler → Simulation abgebrochen |
| **Live** | **Asynchron** (non-blocking im Pipeline-Thread). Events gehen in den Spool-Buffer (Kap 2, Abschnitt 10) | Pipeline-Thread blockiert **nicht**. Non-droppable Events (QuantScore) gehen in den Bounded Spool. Spool-Overflow → `DataQualityEvent(ESCALATE, EVENTLOG_OVERFLOW)` → odin-core entscheidet |

> **Kein Fail-Close im Live-Modus:** Der Pipeline-Thread wartet nicht auf EventLog-Bestaetigungen. Der QuantScore ist fuer die unmittelbare Entscheidung (Arbiter → Risk Gate → OMS) ein In-Memory-Objekt. Die EventLog-Persistierung dient der Forensik und dem Replay — nicht der Entscheidungsfindung selbst. Deshalb ist async + spool + ESCALATE-bei-Overflow die richtige Strategie (konsistent mit Kap 2).

> **Was "non-droppable" im Live-Modus konkret bedeutet:**
> - Das System droppt QuantScore-Events **nicht intentional** (anders als best-effort MarketEvents, die bei Ueberlast gesampelt werden duerfen)
> - QuantScore-Events gehen in den **Bounded In-Memory-Spool** (Default: 10.000 Events, Kap 2). Solange der Spool nicht voll ist, werden sie nachgeliefert
> - Spool > 80%: `WARN`-Metrik. Spool voll: `ESCALATE` → odin-core entscheidet (Kill-Switch oder Degraded-Continue)
> - **Spool-Durability: In-Memory** (Kap 2, Abschnitt 10). Bei Prozess-Crash gehen ungespoolte Events verloren. Das ist akzeptabel fuer Crash-Recovery v1 (nicht restartable intraday, Kap 0) — alles vor dem Crash ist bereits im EventLog persistiert, Post-Crash-Forensik nutzt das bereits Persistierte
> - **Kein Disk-Backed-Spool** in v1 — der Mehrwert waere marginal, da ein Crash ohnehin den Handelstag beendet

---

## 12. Scope und Lifecycle

### Pro-Pipeline-Instanziierung

Die KPI-Engine wird **pro Pipeline** instanziiert (Kap 1, Abschnitt 7). Jede Pipeline hat:
- Eigene ta4j-BarSeries (pro Timeframe)
- Eigene Indikator-Instanzen
- Eigenen Warmup-State

Keine geteilten Indikator-Daten zwischen Pipelines. Die `PipelineFactory` (odin-core) erzeugt die KPI-Engine manuell (kein Spring-Bean, Kap 1 Abschnitt 6).

**Konfigurationsweitergabe:** Spring laedt die `KpiProperties` und `QuantProperties` (Records mit `@ConfigurationProperties`) einmal beim Start. Die PipelineFactory erhaelt diese Records per Constructor Injection und reicht sie an jede KPI-Engine-Instanz weiter. Die KPI-Engine hat keine direkte Spring-Abhaengigkeit.

### Lifecycle

| Phase | KPI-Engine-Verhalten |
|-------|---------------------|
| **INITIALIZING** | BarSeries werden erzeugt (leer). Keine Berechnung, kein Decision-Cycle |
| **WARMUP → ACTIVE** | Decision-Cycle laeuft ab dem ersten MarketSnapshotEvent. KPI-Engine erzeugt immer IndicatorResult. `warmupComplete` wechselt automatisch zu `true` sobald alle Indikator-Perioden gefuellt sind (typisch nach 21+ Decision-Bars und 14+ 5m-Bars). Waehrend `warmupComplete = false`: Rules Engine erzeugt keinen Intent → Quant/Arbiter werden nicht aufgerufen (siehe Abschnitt 2, Warmup-Verhalten) |
| **ACTIVE** | Volle Berechnung. QuantScore wird bei jedem Trade-Intent erzeugt |
| **EOD** | Letzter IndicatorResult. Danach kein weiterer Zugriff. BarSeries werden beim naechsten SOD geleert |

> **Kein separater WARMUP-State in der KPI-Engine:** Die KPI-Engine hat keinen eigenen State fuer Warmup. `warmupComplete` ist ein berechneter Boolean im IndicatorResult. Die Pipeline-State-Machine (PipelineStateMachine in odin-core) steuert den uebergeordneten Lifecycle — die KPI-Engine reagiert nur auf MarketSnapshotEvents.

### Thread-Modell

Die KPI-Engine laeuft **synchron im Pipeline-Thread** (Kap 0, Abschnitt 6: Decision Loop strikt sequentiell). Keine eigenen Threads, kein Thread-Pool.

**Begruendung:** Bei 2 Timeframes × ~7 Indikatoren × < 1ms pro Indikator ist die Gesamtdauer < 15ms — weit unter der Bar-Periode von 180s (3m) bzw. 300s (5m). Parallelisierung waere premature Optimization (Kap 0, Leitprinzip Lean).

> **Spaetere Optimierung:** Falls Profiling einen Bottleneck belegt (z.B. bei Erweiterung auf > 10 Timeframes), kann die Berechnung auf einen Fork-Join-Pool verlagert werden. Die API (synchroner Return von IndicatorResult) aendert sich dabei nicht.

---

## 13. Konfiguration

```properties
# odin-brain.properties (KPI-Abschnitt)

# Indikator-Perioden
odin.kpi.ema-short-period=9
odin.kpi.ema-long-period=21
odin.kpi.rsi-period=14
odin.kpi.atr-period=14
odin.kpi.bollinger-period=20
odin.kpi.bollinger-std-dev=2.0
odin.kpi.adx-period=14

# BarSeries-Kapazitaet (Decision-Bars: 130 bei 3m, 78 bei 5m)
odin.kpi.barseries.capacity-decision=130
odin.kpi.barseries.capacity-5m=78

# Quant-Schwellenwerte
odin.kpi.quant.score-threshold=0.50
odin.kpi.quant.high-confidence-threshold=0.70
odin.kpi.quant.veto-spread-max-percent=0.5
odin.kpi.quant.veto-rsi-max=75
odin.kpi.quant.veto-vwap-deviation-max-percent=2.0

# Gewichtungen
odin.kpi.quant.weight-trend=0.25
odin.kpi.quant.weight-rsi=0.20
odin.kpi.quant.weight-volume=0.20
odin.kpi.quant.weight-spread=0.15
odin.kpi.quant.weight-rr-ratio=0.20
```

> **Kein `odin.kpi.compute-threads`:** Sequentielle Berechnung im Pipeline-Thread (Abschnitt 12). Kein Thread-Pool-Config in v1.

---

## 14. Abhaengigkeiten und Schnittstellen

### Konsumiert

- `MarketSnapshotEvent` (von odin-data, publiziert nach Bar-Close) — Trigger fuer KPI-Berechnung
- `MarketSnapshot.decisionBars`, `MarketSnapshot.fiveMinBars` — Bars fuer ta4j-BarSeries
- `MarketSnapshot.vwap` — VWAP als Source-of-Truth aus odin-data (Kap 2)
- `MarketSnapshot.spreadStats` — SpreadStats fuer Quant Validation (Kap 2)
- `MarketSnapshot.currentPrice`, `MarketSnapshot.currentBid`, `MarketSnapshot.currentAsk` — Preis-Inputs
- `MarketSnapshot.flags` — DataFlags (insbesondere `BAR_APPROXIMATED`)
- `MarketSnapshot.runId`, `MarketSnapshot.instrumentId`, `MarketSnapshot.marketTime` — Join-Keys
- `MarketClock` — Zeitreferenz (indirekt via MarketSnapshot.marketTime)
- `EventLog` — Persistierung von IndicatorResult (best-effort) und QuantScore (non-droppable)
- `LlmAnalysis.regimeConfidence` — Fuer dynamische Quant-Schwelle (Abschnitt 8)

### Produziert

- `IndicatorResult` (DTO in odin-api) — Alle berechneten Indikator-Werte + Flags + warmupComplete
- `QuantScore` (DTO in odin-api) — Gesamtscore + Einzel-Checks + Veto-Flags + approved-Flag

### Nicht in Scope

- VWAP-Berechnung → `odin-data` (Kap 2)
- SpreadStats-Berechnung → `odin-data` (Kap 2)
- Intraday-Extrema → `odin-data` (Kap 2)
- Pattern-State-Machines → `odin-brain` RulesEngine (Kap 6)
- Decision-Kombination → `odin-brain` DecisionArbiter
- Position Sizing → `odin-execution` (Kap 7)
