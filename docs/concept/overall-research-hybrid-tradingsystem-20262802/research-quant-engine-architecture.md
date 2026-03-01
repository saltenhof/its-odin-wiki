# Research: Quantitative Intraday-Trading-Engine-Architektur — State of the Art

> **Stand:** 2026-02-28
> **Scope:** Rein quantitativer Teil — kein LLM/KI, keine Hybrid-Kombination
> **Basis:** 40+ Suchen, akademische Paper, Praxis-Blogs, Open-Source-Implementierungen

---

## Executive Summary (8 Kernerkenntnisse)

1. **Indikator-Redundanz ist das grösste strukturelle Problem.** RSI, MACD und Stochastic messen dasselbe (Momentum). Wer alle drei einsetzt, addiert kein Signal, sondern Rauschen. ODAINs 7-Gate-Kaskade (SPREAD, VOLUME, RSI, EMA, VWAP, ATR, ADX) ist konzeptionell richtig, weil sie orthogonale Informationsquellen kombiniert — Marktqualität (Spread), Volumen, Momentum (RSI), Trend (EMA), Equilibrium (VWAP), Volatilität (ATR), Trendstärke (ADX).

2. **VWAP ist akademisch bestätigt als primärer Intraday-Anker.** Die Zarattini/Aziz-Studie (2023) zeigt Sharpe Ratio 2.1 und 671% Return für eine reine VWAP-Trendfolge-Strategie (QQQ, 2018–2023). Institutionelle Investoren abwickeln ca. 50% aller Orders als VWAP-Execution. ODINs VWAP als Source-of-Truth ist keine Stilentscheidung, sondern empirisch fundiert.

3. **Intraday Momentum der ersten Handelsstunde ist statistisch signifikant.** Gao, Han, Li & Zhou (2018) belegen: Die Erste-Halbe-Stunde-Rendite prediktiert die Letzte-Halbe-Stunde-Rendite mit R² = 1.6–2.6%, generiert Certainty-Equivalent-Gains von 6% p.a. ODINs Opening-Pattern-Gate (UPPER_HALF / LOWER_HALF) hat damit eine solide akademische Grundlage.

4. **Walk-Forward ist notwendig, aber nicht hinreichend.** Kombinatorial Purged Cross-Validation (López de Prado) übertrifft klassisches Walk-Forward bei der Vermeidung von Backtest-Overfitting signifikant (niedrigerer PBO, besserer DSR). ODINs Backtesting-Framework sollte mindestens Parameter-Sensititivitätstests (Neighbor-Jitter) und Deflated-Sharpe-Ratio-Bewertung integrieren.

5. **ATR-Einfrieren bei Entry ist best practice.** Die Praxis-Literatur bestätigt: Entry-ATR für Stop-Placement fixieren verhindert, dass eine nachfolgende Volatilitätsreduktion den Trailing Stop unnötig verengt. ODINs `entryATR frozen`-Ansatz ist korrekt. Empfohlener Multiplikator für Intraday: 1.5–2.5×, nicht 3–4× (die für Swing Trading geeignet sind).

6. **Regime-Detection mit Hidden Markov Models übertrifft statische Regeln.** Eine 3-State-HMM-Strategie erreichte Sharpe Ratio 1.9 auf 5-Minuten-Bars und reduzierte Max-Drawdown von 56% auf 24% gegenüber einem Benchmark. ODINs deterministische Regime-Logik (EMA/ADX/VWAP-Schwellen) ist zuverlässig, könnte aber durch probabilistische Regime-Zustandswahrscheinlichkeiten ergänzt werden.

7. **Signal-Kalibrierung (Probability Calibration) ist weitgehend ungelöst für Trading-Signale.** Was in ML als Platt Scaling und Isotonic Regression bekannt ist, findet im Quant-Trading kaum direkte Anwendung. ODINs `regimeConfidence`-Schwellen (0.5 / 0.7) sind heuristische Approximationen — eine datengetriebene Kalibrierung würde die Genauigkeit erhöhen.

8. **Partial Profit Taking (R-Stufentabelle) ist konservativ korrekt.** Die Literatur zeigt: Partieller Exit bei 1R–2R und Trailing des Rests erzielt besseres Risikoprofil als Alles-oder-Nichts. ODINs R-Floor-Profit-Protection mit `MODERATE`/`AGGRESSIVE`/`RELAXED`-Profilen entspricht dem Stand der Praxis.

---

## 1. Indikator-Selektion und Multi-Faktor-Modelle

### 1.1 Akademische Befunde zur Indikator-Effektivität

Die Forschungsliteratur zeigt ein differenziertes Bild zur Effektivität technischer Indikatoren bei Intraday-Trading:

**RSI (Relative Strength Index)**
- Primäre Stärke auf Tages-Charts; auf Intraday deutlich mehr Fehlsignale
- Optimale Period auf 5-Minuten-Bars: 9–10 (nicht 14) für höhere Responsivität
- Optimale Schwellen auf 5-Min-Bars: 75/25 oder 80/20 statt Standard 70/30 (weniger Fehlsignale)
- RSI + MACD kombiniert: Win Rate 0.84–0.86 gegenüber einzelnen Indikatoren
- Hochvolatile Perioden: RSI erreicht Accuracy-Raten bis 97% (ResearchGate, 2024)

**MACD**
- Alleine: Accuracy ca. 52% (86 von 166 Signalen)
- Deutlich besser in Kombination mit RSI und VWAP
- Keine besondere Eignung für Hochfrequenz-Intraday (minutengenaue Signals)

**ADX (Average Directional Index)**
- Auf Intraday-Charts tendenziell mehr Rauschen als auf Tages-Charts
- Empfohlene Periode für Scalping/Day-Trading: 7–10 statt Standard 14
- Schwelle für "Trend vorhanden": >20 auf 1–15-Min-Charts (nicht >25 wie auf Daily)
- Muss mit Preis-Action kombiniert werden — allein keine Entry/Exit-Signale

**Bollinger Bands**
- Regime-abhängiges Verhalten: Breakout-Strategien im Trend-Regime überlegen, Mean-Reversion im Range-Regime (Arda, SSRN 2025)
- Squeeze-Detection (BBW < 4%): Positive Expectancy über 5–10 Bars nach Ausbruch, Directional Accuracy 55–60%
- Dual-Confirmation mit Keltner-Channel: Bollinger innerhalb Keltner = starke Compression-Bestätigung
- Optimale Intraday-Settings: 10-Perioden, 1.5–2.0 SD (Standard-Settings für 5–15-Min-Charts)

**Volume / Volume Ratio**
- Volumetric Indicators zählen zu den wichtigsten Features
- Normalisiertes Volumen (Volume-Ratio zu SMA-20) besser als absolutes Volumen
- Volumen-Spike (>2×) als Bestätigungssignal für Pattern-Transitions empirisch belegt
- Volume Climax im Uptrend = Erschöpfungssignal, im Downtrend = Boden-Indikator

### 1.2 Feature-Selektion und Redundanz-Analyse

Die Forschung zeigt: Feature-Selektion ist entscheidend. Eine Studie mit 124 technischen Indikatoren reduzierte den Feature-Space durch Redundanz-Elimination erheblich (PMC, 2023):

- Random Forest eignet sich zur Redundanz-Elimination (entfernt hochkorrelierte Features)
- Beste Sharpe Ratios meist mit nur Top-5 bis Top-10 Indikatoren
- Mean Decrease Accuracy (MDA) als Feature-Importance-Methode bevorzugt
- Performance stark abhängig von der korrekten Indikator-Auswahl

**Klassen technischer Indikatoren (Redundanz-Karte):**

| Klasse | Indikatoren | Messen |
|--------|-------------|---------|
| Momentum/Oszillatoren | RSI, Stochastic, CCI, Williams %R | Gleiche Information — nur einen wählen |
| Trend-Following | EMA, SMA, MACD | Teilweise redundant; EMA reagiert schneller |
| Volatilität | ATR, Bollinger-Width, Keltner | Komplementär (ATR absolute, BB relative) |
| Trendstärke | ADX, DMI (+DI/-DI) | Ergänzt EMA-Crossover mit Stärkeinfo |
| Volume | OBV, Volume Ratio, VWAP | VWAP dominierend für Intraday |

**ODIN-Bewertung:** Die Indikator-Selektion in ODINs KPI-Engine (EMA, RSI, ATR, Bollinger, ADX, VWAP, Volume-Ratio) deckt jede Klasse mit genau einem Vertreter ab — dies entspricht dem akademischen Best Practice der Redundanz-Vermeidung.

### 1.3 Information Coefficient (IC) für Indikator-Gewichtung

Der Information Coefficient misst die Korrelation zwischen prognostiziertem und tatsächlichem Return:

- Typische empirische ICs: 0.02–0.10
- IC > 0.05 gilt als stark
- IC-basierte dynamische Gewichtung übertrifft statische Gewichtung deutlich
- Spearman-Rank-IC ist robuster gegenüber Ausreissern als Pearson-IC
- IC-Decay: Kurzfristige Indikatoren (RSI, Volumen) haben schnelleren IC-Verfall als langfristige (ADX)

---

## 2. Signal-Aggregation und Scoring

### 2.1 Aggregationsansätze im Vergleich

**Gewichtetes Scoring**
- Lineare Kombination normierter Indikator-Scores
- Vorteil: Graduell, kein harter Cutoff
- Nachteil: Schlechte Indikatoren "verwässern" gute
- Empfehlung: Z-Score-Normalisierung pro Indikator, dann gewichtete Summe

**Voting (Mehrheitsentscheidung)**
- N Indikatoren, Mehrheit entscheidet
- Einfach, robust, aber binär
- Geeignet für Entry-Entscheidung als Endschicht

**Gate-Kaskaden (Sequential Filtering)**
- Hard-Veto-Gates eliminieren ungeeignete Zustände zuerst
- Short-Circuit verhindert unnötige Berechnungen
- ODIN-Ansatz: 7 Gates in fester Prioritätsreihenfolge
- Akademisch: Keine direkte Literatur zu Gate-Kaskaden als Konzept, aber Sequential-Filtering ist in der Signal-Verarbeitungsliteratur als "Coarse-to-Fine"-Strategie bekannt
- Vorteil gegenüber reinem Scoring: Harte Constraints werden nicht durch gute andere Signale überstimmt

**Z-Score-basierte Spread-Normalisierung**
- Normalisierung über rolling Mean/Std: `z = (value - mean) / std`
- Bull-Bear Spread: `normalized_bull - normalized_bear`
- Praktisch: Composite Bull-Bear Dominance Index (TradingView-Community)
- Vorteil: Eliminiert Niveau-Unterschiede zwischen Indikatoren auf unterschiedlichen Skalen

### 2.2 Confidence-Kalibrierung

Die Frage "Was bedeutet ein Confidence-Score von 0.8?" ist in der ML-Literatur als Probability Calibration bekannt:

- **Platt Scaling** (Logistic Regression auf Rohscores): Gut bei wenig Daten, parametrisch
- **Isotonic Regression**: Nicht-parametrisch, besser bei grossen Datensätzen
- **Reliability Diagrams**: Visualisieren Kalibrierungsgüte (gut kalibriert = Diagonale)

Für Trading: Ein Regime-Confidence-Score von 0.8 sollte empirisch bedeuten, dass das Regime in 80% der Folgeperioden tatsächlich anhält. ODINs Schwellen (0.5/0.7) sind sinnvolle Heuristiken, aber keine kalibrierten Wahrscheinlichkeiten.

**Empfehlung:** Retrospektive Kalibrierungsanalyse im Backtest: Für jedes `regimeConfidence`-Band messen, wie oft das Regime tatsächlich N Bars anhielt.

### 2.3 Multi-Factor Signal-Qualität

Forschungsbefunde zur Kombination:
- VWAP + RSI + MACD: Win Rate 0.84–0.86 (deutlich besser als Einzelsignale)
- Multi-Timeframe-Alignment erhöht Win Rate von 39% auf 58% (Trade-aligned vs. non-aligned)
- 5 Simultane Faktoren aus orthogonalen Klassen: Optimal für die meisten Strategien

---

## 3. Entry/Exit-Frameworks

### 3.1 Entry-Strategien: Was funktioniert

**Opening Range Breakout (ORB)**
- Zarattini/Aziz (2024): 5-Minuten-ORB bestes Ergebnis, Sharpe Ratio 2.4, beta nahezu 0
- Andere Implementierung: Sharpe 2.81, Profit Factor 1.23, Max Drawdown 18%
- Einschränkung: Einige Quants berichten sinkende Effektivität in modernen Märkten
- Alignment mit ODIN: ODINs `OpeningConsolidationFsm` entspricht diesem Konzept

**VWAP-Trendfolge**
- Zarattini/Aziz (2023): Long wenn Preis > VWAP, Short wenn Preis < VWAP
- QQQ 2018–2023: 671% Return, Sharpe 2.1, Max Drawdown 9.4%
- Einfachste Form, akademisch robust belegt
- Alignment mit ODIN: VWAP-Gate in Gate-Kaskade, VWAP-Proximity in Entry-Rules

**Intraday Momentum der ersten Halbe-Stunde**
- Gao, Han, Li, Zhou (Journal of Financial Markets, 2018): Erste-30-Min-Return prediktiert Letzte-30-Min-Return
- R² = 1.6%, Certainty-Equivalent-Gains ~6% p.a.
- Stärker an volatilen Tagen und Makro-News-Tagen
- Global bestätigt: Li, Sakkas, Urquhart (Journal of Financial Markets, 2022) — 16 Märkte
- Alignment mit ODIN: ODINs Opening-Pattern-Gate (UPPER_HALF/LOWER_HALF) ist die direkte Umsetzung

**Support/Resistance Bounce-Entries**
- Algorithmische S/R-Detektion: Kernel Density auf Preis-Ticks, Agglomerative Clustering auf Pivot-Punkten
- Mathematisch fundiert: MDPI 2022 — S/R-Levels verbessern Profitabilität von Algo-Modellen
- Multiple-Bounce-Eskalation: Jeder bestätigte Bounce erhöht die Konfidenz
- Für High-Beta-Aktien: Toleranzband typisch 0.1–0.3% (sehr scharf)
- Alignment mit ODIN: `BounceEvent`-System mit `bounceConfidenceMin` und VWAP-Bypass

### 3.2 Trailing Stop Strategien

**Chandelier Exit (Charles Le Beau / Alexander Elder)**
- Formel: `Stop = Highest_High(n) - Multiplier × ATR(n)` (für Long)
- Default: 22 Perioden (für Tages-Charts); Intraday: 10–14 Perioden empfohlen
- Vorteil: Kein vorzeitiges Exit bei normaler Volatilität
- Nachteil: Als Entry-Signal zu viele False Signals; nur als Stop-Loss geeignet

**ATR Trailing Stop (Highwater-Mark-Variante)**
- Beste Variante für Intraday-Trading
- Stop = max(StopPrevious, HighwaterMark - Multiplier × ATR)
- Highwater-Mark-Prinzip: Stop kann nie sinken — verhindert Lock-in-Paradox
- Multiplier für Intraday: 1.5–2.5× (Daily/Swing: 3–4×)
- Entry-ATR einfrieren: Verhindert Verwässerung des Stops durch spätere Volatilitätsreduktion
- Alignment mit ODIN: Exakt so implementiert (`entryATR frozen`, Highwater-Mark)

**EMA-basierter Trailing Stop**
- Alternative: Stop = EMA(N) × (1 - Buffer%)
- Vorteil: Glatter, weniger sensitiv auf einzelne Bars
- Nachteil: Reagiert langsamer auf Trendbrüche

### 3.3 R-basiertes Profit-Protection Framework (Van Tharp)

Van Tharp popularisierte das R-Multiple-System:
- `1R = Initial Risk = Entry - Initial Stop`
- Partieller Exit bei 1.5R oder 2R, Trail des Rests
- Typische Stufentabelle (van Tharp / ODIN MODERATE):
  - 1.0R: Break-Even-Stop
  - 1.5R: Stop auf +0.5R
  - 2.0R: Stop auf +1.0R
  - 3.0R+: Aggressiver Trail

Akademische Basis: Partial Profit Taking erzielt bessere Risk-Adjusted-Returns als vollständiger Exit. Emotionale Komponente: Sicherung eines Teils reduziert Holding-Druck für den Runner.

### 3.4 Triple Barrier Method (López de Prado)

Akademischer Ansatz für Labeling und Exit-Design:
- Drei Barrieren: Take-Profit (oben), Stop-Loss (unten), Zeitbarriere (vertikal)
- Label: +1 wenn TP zuerst erreicht, -1 wenn SL zuerst, 0 wenn Zeit zuerst
- Spiegelt echtes Trading-Verhalten besser als Next-Bar-Prediction
- Alignment mit ODIN: ODINs Exit-Priorisierung (Stop-Loss Prio 3, EOD Prio 2, Trailing-Stop Prio 4) entspricht dem Drei-Barrieren-Konzept

### 3.5 Pattern-State-Machines: Akademische Grundlagen

FSM für Preis-Pattern-Erkennung ist akademisch belegt (IEEE, 2014 — FSM-FT für Pattern Recognition). Neuere Ansätze nutzen Deep Learning, aber FSMs bleiben für explizite, interpretierbare Patterns die beste Wahl:

- Interpretable Trading Pattern designed for ML (ScienceDirect, 2023)
- 3.518 extrahierte Chart-Formationen mit Predictive Accuracy 60–85%
- Momentum-Flush-Recovery-Patterns: In der akademischen Literatur nicht unter dem ODIN-Namen, aber das Prinzip "Oversold-Bounce nach starkem Abwärtssweep" ist in der Intraday-Reversal-Literatur dokumentiert (NYU Tilburg, 2016)

---

## 4. Risk Management

### 4.1 Position Sizing: Methoden-Vergleich

| Methode | Beschreibung | Vorteil | Nachteil |
|---------|-------------|---------|---------|
| Fixed Fractional | Fester % des Kapitals pro Trade | Einfach, kompoundiert gut | Ignoriert Volatilität |
| Kelly Criterion | Mathematisch optimale Bet-Grösse | Maximiert geometrisches Wachstum | Zu aggressiv, Estimation-Error problematisch |
| Half-Kelly | 50% des Kelly-Optimums | ~75% weniger Volatilität bei ~25% Wachstumsverlust | Konservativ, aber praktikabel |
| Volatility-Targeting | ATR-basiert: grössere ATR = kleinere Position | Passt Risikoexposition an Marktbedingungen | Braucht ATR-Referenzwert |
| R-basiert (Van Tharp) | Fixiertes $ pro 1R | Direkt risk-orientiert, intuitiv | ATR-Abhängig für Stop-Distanz |

**Best Practice für Intraday:** Kombination aus R-basiertem Sizing und Volatility-Scaling:
1. Definiere Max-Risk pro Trade (z.B. 0.5% des Kapitals = 1R)
2. Berechne ATR-Distanz für Stop
3. Position Size = Max-Risk / (ATR × Multiplikator)

Alignment mit ODIN: ODINs Risk-Gate implementiert ATR-basiertes Sizing mit `SizeModifier`-Faktor vom Arbiter — entspricht dem Best Practice.

### 4.2 Portfolio-Risk bei Multi-Instrument-Trading

Forschungsbefunde (2023):
- Hochkorrelierte Instrumente: Korrelation von 5% Marktbewegung kann 15–20% Account-Drawdown erzeugen
- Korrelations-Timing: Wenn Strategien zeitgleich Drawdown erleiden → kumulatives Risiko
- Diversifikationsregel: Intraday-Positionen in unkorrelierten Sektoren/Regimen anstreben
- Empfehlung: Max 2% Kapitalexposition pro korreliertem Sektor-Cluster

Alignment mit ODIN: ODINs GlobalRiskManager verwaltet Account-Limits über alle Pipelines — strukturell korrekt. Explizite Korrelations-Berechnung zwischen simultanen Positionen fehlt noch.

### 4.3 Tägliche Verlustlimits

Best Practice:
- Daily Loss Limit: 1.5–3.0% des Eigenkapitals (Daumenregel: ~3 verlorene 1R-Trades)
- EOD-Trailing-Drawdown: Limit bezieht sich auf Intraday-High des Tages, nicht auf Eröffnungskapital
- Intraday-Halt bei Erreichen: Keine weiteren Trades am selben Tag

Alignment mit ODIN: `DAY_STOPPED`-State in der Pipeline-FSM + globaler `AccountRiskState` deckt dies ab.

---

## 5. Regime-Erkennung

### 5.1 Deterministische Regime-Erkennung (Regelbasiert)

ODINs Ansatz mit EMA-Kreuzung + ADX + VWAP-Relation ist eine klassische regelbasierte Regime-Klassifikation. Vorteile:
- Vollständig interpretierbar
- Keine historischen Daten für Training nötig
- Kein Look-Ahead-Bias

Schwächen:
- Scharfe Regime-Grenzen (binär statt graduell)
- Keine Unsicherheitsquantifizierung
- Hysterese (2-Bar-Confirmation) hilft, ist aber ebenfalls heuristisch

### 5.2 Hidden Markov Models (HMM) für Regime-Erkennung

Die HMM-Literatur (QuantConnect, QuantStart, MDPI 2020) zeigt starke Ergebnisse:

- 3-State-HMM (Bull, Neutral, Bear) auf 5-Minuten-Bars: Sharpe Ratio 1.9
- Max-Drawdown-Reduktion: Von 56% auf 24% vs. Benchmark
- HMMs modellieren inhärente Regime-Persistenz (Übergänge nicht instantan)
- Ausgabe: Regime-Wahrscheinlichkeitsverteilung statt harte Klassifikation

**Implementierungsprinzip:**
1. HMM auf Return-Sequenzen trainieren (unsupervised)
2. Viterbi-Algorithmus für aktuellen Regime-State
3. Forward-Algorithmus für Regime-Wahrscheinlichkeiten
4. Strategieparameter per Regime anpassen

**Vorteil gegenüber ODIN-Ansatz:** HMM liefert `P(Regime=TREND | Observations)` als kontinuierliche Zahl — direkt als `regimeConfidence` verwendbar.

### 5.3 Statistical Jump Models (Neuere Forschung)

Aydinhan, Kolm, Mulvey, Shu (Annals of Operations Research, 2024):
- Erweiterung von HMMs: Jump-Penalty verhindert zu häufige Regime-Wechsel
- Kontinuierliche Jump Models (CJM): Regime-State als Wahrscheinlichkeitsvektor
- Signifikant bessere Out-of-Sample-Performance als klassische HMMs
- Anwendung auf US/DE/JP-Märkte 1990–2023 mit Transaktionskosten

### 5.4 Regime-Hysterese: Empirische Grundlage

Das Hysterese-Prinzip (N Bars Bestätigung vor Regime-Wechsel) ist in der Literatur als notwendige Massnahme gegen Whipsawing bestätigt:

- Zu kurze Hysterese: Zu viele Wechsel, hohe Transaktionskosten
- Zu lange Hysterese: Regime-Wechsel zu spät erkannt
- Empfehlung: 2–3 Bars auf 5-Min-Basis; tagesübergreifend nicht notwendig (Intraday-Reset)

Alignment mit ODIN: `regimeChangeConfirmations = 2` ist im Zielbereich.

### 5.5 Regime-abhängige Parametrisierung

Research bestätigt: Adaptive Parameter je Regime sind überlegen:

- Volatile Regime: Engere Stops, kleinere Positionen, weniger Entry-Filter
- Trend-Regime: Moderatere Stops, normale Positionen, VWAP-Trend-Alignment
- Range-Regime: VWAP-Support-Einträge, schnellere Gewinnmitnahme

Alignment mit ODIN: Regime-spezifische Entry-Conditions (TREND_UP vs. RANGE_BOUND vs. HIGH_VOLATILITY) sind implementiert — entspricht dem Best Practice.

---

## 6. Backtesting und Robustheit

### 6.1 Walk-Forward-Optimierung

Prinzip: Trainiere auf einem Fenster, teste auf dem nächsten, rolle vor, wiederhole.

**Algorithmus:**
1. Definiere In-Sample-Fenster (z.B. 90 Handelstage)
2. Optimiere Parameter auf In-Sample
3. Teste auf Out-of-Sample-Fenster (z.B. 30 Handelstage)
4. Rolle beide Fenster vor
5. Aggregiere Out-of-Sample-Performance

**Schwäche:** Zeitliche Variabilität, einzelne "Glücks-Epochen" können Ergebnis verzerren.

### 6.2 Combinatorial Purged Cross-Validation (CPCV)

López de Prado & Bailey (2014+) entwickelten CPCV als robustere Alternative:

- Teilt Zeitreihe in N nicht-überlappende Gruppen
- Generiert alle k-aus-N Kombinationen als Test-Sets
- **Purging:** Entfernt Training-Samples die Test-Zeitraum überlappen
- **Embargo:** Sicherheitsabstand zwischen Training und Test verhindert Information-Leakage
- Ausgabe: Verteilung von Performance-Metriken (nicht ein einzelner Wert)
- Nachweis (ScienceDirect, 2024): CPCV signifikant überlegen bei PBO (Probability of Backtest Overfitting)

**Empfehlung für ODIN:** CPCV als primäre Backtesting-Methode für Parameter-Optimierung.

### 6.3 Deflated Sharpe Ratio (DSR)

Bailey & López de Prado (2014) entwickelten DSR als Korrektur für:
- **Multiple Testing Bias:** Viele Parameter-Kombinationen → eine findet zufällig gutes Ergebnis
- **Non-Normalität:** Intraday-Returns sind typisch leptokurtisch (Fat Tails)
- **Kürzere Sample-Längen:** Kürzere Backtests haben grössere Schätzfehler

DSR berechnet: P(SR > SR_Benchmark | n_trials, sample_length, skewness, kurtosis)

**Empfehlung für ODIN:** DSR als primäre Erfolgskennzahl für Backtesting. Ein Sharpe Ratio von 1.5 aus 500 Parameter-Kombinationen ist wertlos ohne DSR-Korrektur.

### 6.4 Monte Carlo Simulation für Robustheitstests

Fünf Monte Carlo Methoden (StrategyQuant):
1. **Trade-Order-Shuffle:** Zufällige Reihenfolge der Trades
2. **Parameter Jitter:** Kleine Änderungen der Parameter
3. **Trade Omission:** Zufälliges Weglassen von x% der Trades
4. **Slippage Variation:** Zufällige Slippage-Variationen
5. **Walk-Forward Monte Carlo:** Kombination aus WFO und MC

Robustes System: Leistung bleibt stabil unter allen 5 Methoden.

### 6.5 Parameter-Stabilitäts-Test (Neighbor Test)

Best Practice (Ehlers, MesaSoftware):
1. Führe Grid-Search über alle relevanten Parameter aus
2. Finde optimale Kombination
3. Teste alle Nachbar-Kombinationen (±10% pro Parameter)
4. Robustes System: Nachbarn zeigen ähnliche Performance (keine steilen Klippen)
5. Roter Flag: Optimum ist eine "Insel" umgeben von viel schlechteren Ergebnissen

**Empfehlung für ODIN:** Vor jeder Parameter-Änderung Neighbor-Test durchführen.

### 6.6 Bias-Vermeidung

Kritische Biases im Backtesting:

| Bias | Beschreibung | Prävention |
|------|-------------|-----------|
| Survivorship Bias | Nur überlebende Aktien in historischen Daten | Point-in-Time-Daten; delisted Aktien einschliessen |
| Look-Ahead Bias | Zukünftige Daten im Signal verwendet | Strikte Event-basierte Simulation; kein `future()` |
| Overfitting | Parameter zu eng auf historische Daten angepasst | CPCV + DSR + Monte Carlo |
| Transaction Costs | Slippage, Spread, Kommission ignoriert | Konservative Slippage-Schätzung (0.5× Spread) |

Survivorship Bias in US-Aktien: Überbewertet historische Renditen um ca. 1–4% p.a.

Alignment mit ODIN: ODINs Backtesting (`BacktestRunner`) ist event-basiert — Look-Ahead kein strukturelles Problem. Survivorship Bias bei Instrument-Auswahl muss manuell vermieden werden.

---

## 7. ATR-basierte Parametrisierung

### 7.1 ATR als universeller Skalierungsfaktor

ATR ist der am weitesten verbreitete Volatilitäts-Skalar im Intraday-Trading. Anwendungsfelder:

- **Stop-Loss-Distanz:** `stop = entry - n × ATR`
- **Position-Sizing:** `size = risk_$ / (n × ATR × price_per_unit)`
- **Target-Setting:** `target = entry + m × ATR` (m > n für positive R/R)
- **Band-Breiten:** `band_width = 2 × ATR` für dynamische Konsolidierungszonen
- **Volatility-Regime:** Vergleich aktueller ATR mit gleitendem ATR-Durchschnitt

### 7.2 ATR-Periodenauswahl für Intraday

Forschungs-Konsens für 5-Minuten-Bars:

| Periode | Eignung |
|---------|---------|
| ATR(5) | Sehr reaktiv, gut für Scalping (1–5 Min Holding) |
| ATR(7–10) | Optimal für Day-Trading mit 5-Min-Bars |
| ATR(14) | Standard für Daily/Swing; auf 5-Min-Bars zu träge |
| ATR(20+) | Nur für übergeordneten Kontext |

Für ODINs 5-Minuten-Bars: ATR(14) ist konservativ. ATR(10) oder ATR(7) würde schneller auf aktuelle Volatilität reagieren.

### 7.3 ATR einfrieren vs. dynamisch nachführen

**Einfrieren bei Entry (ODINs Ansatz):**
- `entryATR` = ATR zum Zeitpunkt der Order-Abgabe
- Stop-Distanz basiert auf diesem Wert
- Vorteil: Stop-Level bleibt konstant, keine unerwarteten Verschiebungen
- Nachteil: Bei stark steigender Volatilität evtl. zu enger Stop

**Dynamisch nachführen:**
- `currentATR` wird kontinuierlich aktualisiert
- Trailing-Stop-Abstand wächst bei Volatilitätsanstieg
- Problem: Stop kann nach unten wandern wenn ATR fällt (Highwater-Mark verhindert dies teilweise)

**Best Practice:** Einfrieren für `stopLoss`-Distanz (Schutz vor zu frühem Stop-Out), dynamisch für `trailingStop`-Multiplikator (passt sich an aktuelles Marktumfeld an). ODINs Ansatz (Entry-ATR eingefroren) ist korrekt.

### 7.4 ATR-Multiplikatoren: Empfohlene Bereiche

| Verwendung | Typischer Multiplier | ODIN |
|-----------|---------------------|------|
| Stop-Loss Intraday | 1.5–2.5× | 2.2× (konfiguriert) |
| Trailing Stop Intraday | 1.5–2.5× | Faktor WIDE/NORMAL/TIGHT |
| Position-Target | 2.5–4× (entspricht 1.5R–2.5R bei 2× Stop) | R-Stufentabelle |
| ATR-Decay-Ratio | Eigenimplementierung | Implementiert |

**ATR-Decay-Ratio** (ODINs Eigenimplementierung): Verhältnis `ATR_aktuell / ATR_SMA(20)` — indiziert ob aktuelle Volatilität über oder unter dem rollenden Durchschnitt liegt. Sinnvoll als Regime-Signal (HIGH_VOLATILITY-Erkennung).

### 7.5 ATR und Volatility-Contraction für Breakout-Detection

Aus der Coil/Compression-Literatur:
- ATR an Recent-Low = Kompressionsphase, hohes Breakout-Potential
- Dual-Confirmation: ATR niedrig UND Bollinger innerhalb Keltner = starkes Squeeze-Signal
- TTM-Squeeze-Indikator formalisiert dies: Bollinger + Keltner + Momentum-Histogram
- Alignment mit ODIN: `CoilBreakoutFsm` nutzt fallende Highs + steigende Tiefs als Kompressions-Signal — ATR-Integration als Bestätigung wäre ergänzende Verbesserung

---

## Vergleich mit ODIN Ist-Zustand

### Was ODIN bereits sehr gut macht

| Aspekt | ODIN-Umsetzung | Bewertung |
|--------|---------------|-----------|
| Indikator-Selektion | 7 Indikatoren aus 6 orthogonalen Klassen | Optimal — keine Redundanz |
| VWAP als Source-of-Truth | Konsumiert aus odin-data, nicht nachberechnet | Best Practice |
| ATR einfrieren bei Entry | `entryATR` fixiert bei Order | Best Practice |
| Highwater-Mark | Trail kann nie sinken | Korrekt |
| R-Stufentabelle | MODERATE/AGGRESSIVE/RELAXED | Entspricht Praxis-Standard |
| Regime-Hysterese | 2-Bar-Confirmation | Im empfohlenen Bereich |
| Gate-Kaskade (7 Gates) | Orthogonale Filterung in Prioritätsreihenfolge | Konzeptuell sehr gut |
| Opening-Pattern-Gate | UPPER_HALF / LOWER_HALF-Bestätigung | Akademisch belegt (Intraday-Momentum-Literatur) |
| Exhaustion-Detektor | 3-Säulen-Modell (Extension/Climax/Rejection) | Theoretisch fundiert |
| R/R-Prüfung | Im Risk-Gate | Standard |
| Multiple-Bounce-Eskalation | BounceEvent + `bounceConfidenceMin` | Entspricht S/R-Literatur |

### Identifizierte Gaps

| Gap | Beschreibung | Priorität |
|-----|-------------|-----------|
| Backtest-Validierung | Kein CPCV, kein DSR, kein Monte Carlo | Hoch |
| ATR-Periode für Intraday | ATR(14) auf 5-Min möglicherweise zu träge | Mittel |
| Regime-Wahrscheinlichkeiten | Binäre Regime statt probabilistisch (HMM) | Mittel |
| Korrelations-Management | Keine explizite Korrelation zwischen parallelen Positionen | Mittel |
| Signal-Kalibrierung | `regimeConfidence`-Schwellen heuristisch | Niedrig |
| Volume Profile (VPOC/VAH/VAL) | Nicht integriert | Niedrig |
| ADX-Periode Intraday | ADX(14) evtl. zu träge für 5-Min-Intraday (besser: ADX(7)) | Niedrig |
| ATR-Periode für Intraday | ATR(14) vs. ATR(10) für schnellere Responsivität | Niedrig |

---

## Empfehlungen

### Empfehlung 1: Backtesting-Framework um CPCV und DSR erweitern

**Was:** Integriere Combinatorial Purged Cross-Validation und Deflated Sharpe Ratio als primäre Backtesting-Methoden.

**Warum:** ODINs Backtest-Ergebnisse sind aktuell einem unkorrigierten Multiple-Testing-Bias ausgesetzt. Jede Parameter-Optimierung (z.B. der `stopLossAtrFactor`) produziert einen inflationierten Sharpe Ratio, wenn kein DSR-Test durchgeführt wird. Praxis-Studie: 95% aller getesteten systematischen Strategien schlagen im Live-Betrieb den Backtest nicht.

**Konkret:** Implementiere Backtest-Robustheitsprüfung als Post-Processing-Schritt:
1. Parameter-Grid über kritische Parameter (ATR-Multiplikatoren, RSI-Schwellen, ADX-Schwellen)
2. Neighbor-Test: Alle ±10%-Variationen des Optimums testen
3. Monte Carlo: 1000 Trade-Order-Shuffles
4. DSR berechnen: Korrigiert für Anzahl Parameter-Kombinationen

### Empfehlung 2: ATR-Periode auf Intraday-Effizienz anpassen

**Was:** Evaluiere ATR(10) oder ATR(7) als Alternative zu ATR(14) für die 5-Minuten-Bars in der KPI-Engine.

**Warum:** ATR(14) auf 5-Min-Bars entspricht 70 Minuten Look-Back — für Intraday-Entscheidungen möglicherweise zu träge. ATR(10) = 50 Minuten, ATR(7) = 35 Minuten, was besser zum Intraday-Zeitfenster passt.

**Vorgehen:** Backtest mit ATR(14) vs. ATR(10) vs. ATR(7) auf denselben historischen Daten. Neighbor-Test beider Werte. Höhere Responsivität ≠ besser (mehr Fehlsignale möglich).

### Empfehlung 3: Volume Profile (VPOC) als S/R-Quelle integrieren

**Was:** Ergänze die S/R-Erkennung um den Volume Point of Control (VPOC) aus dem Session-Volume-Profil.

**Warum:** VPOC ist der Preislevel mit dem meisten Handelsvolumen in der Session. Er fungiert als starkes S/R-Level, an dem institutionelle Trader reagieren. Die "80%-Regel" besagt: Preis der einmal aus der Value Area austritt, hat 80% Wahrscheinlichkeit, zur anderen Seite der Value Area zu rotieren.

**Konkret:** VPOC berechnen aus den Minuten-Bars (kumulatives Volumen pro Preislevel), als S/R-Level in `SrEngine` einspeisenals. Täglicher Reset wie VWAP.

### Empfehlung 4: Regime-Confidence probabilistisch machen

**Was:** Ersetze oder ergänze die heuristische `regimeConfidence`-Schätzung durch ein probabilistisches Modell.

**Warum:** Aktuelle `regimeConfidence` wird aus der Stärke der KPI-Signale abgeleitet (wie weit EMA9 > EMA21 etc.) — nicht aus der tatsächlichen historischen Trefferquote dieser Signalkombinationen. Eine kalibrierte Wahrscheinlichkeit würde präzisere Gate-Entscheidungen ermöglichen.

**Kurzfristig (ohne HMM):** Retrospektive Kalibrierungsanalyse — für jedes `regimeConfidence`-Band messen, wie oft das Regime N Bars anhielt. Schwellen empirisch anpassen.

**Langfristig:** 3-State-HMM auf historische 5-Min-Returns trainieren. Ausgabe: `P(Regime | Observations)` direkt als `regimeConfidence` verwenden.

### Empfehlung 5: Korrelations-Check für Multi-Instrument-Risiko

**Was:** Beim Start einer neuen Position prüfen, ob bestehende offene Positionen hochkorreliert sind.

**Warum:** Bei 2–3 parallelen Positionen in hochkorrelierten Aktien (z.B. beide aus dem gleichen Sektor) kann ein sektorweiter Einbruch alle Positionen gleichzeitig stoppen. Das globale Risiko übersteigt dann das arithmetische Summenrisiko erheblich.

**Konkret:** Im `GlobalRiskManager` oder `RiskGate`: Berechne Rolling-Korrelation der letzten N 5-Min-Returns zwischen allen offenen Instrumenten. Blockiere neuen Entry wenn Korrelation > Schwelle (z.B. 0.7) mit einer bestehenden Position.

---

## Quellen

### Akademische Paper

- [Gao, Han, Li, Zhou — "Market Intraday Momentum" (Journal of Financial Markets, 2018)](https://www.sciencedirect.com/science/article/abs/pii/S0304405X18301351)
- [Li, Sakkas, Urquhart — "Intraday Time Series Momentum: Global Evidence" (Journal of Financial Markets, 2022)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3460965)
- [Zarattini, Aziz — "VWAP The Holy Grail for Day Trading Systems" (SSRN, 2023)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4631351)
- [Zarattini, Aziz — "Can Day Trading Really Be Profitable?" (SSRN, 2023)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4416622)
- [Bailey, López de Prado — "The Deflated Sharpe Ratio" (SSRN, 2014)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2460551)
- [López de Prado — "Advances in Financial Machine Learning" (Buch, 2018)](https://philpapers.org/rec/LPEAIF)
- [Aydinhan, Kolm, Mulvey, Shu — "Identifying Patterns in Financial Markets: Statistical Jump Model" (Annals of Operations Research, 2024)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4556048)
- [Hidden Markov Models Applied to Intraday Momentum (arXiv, 2020)](https://arxiv.org/abs/2006.08307)
- [Backtest Overfitting: Comparison of Out-of-Sample Testing Methods (ScienceDirect, 2024)](https://www.sciencedirect.com/science/article/abs/pii/S0950705124011110)
- [Survey of Feature Selection for Stock Market Prediction (PMC, 2023)](https://pmc.ncbi.nlm.nih.gov/articles/PMC9834034/)
- [Support Resistance Levels Towards Profitability (MDPI, 2022)](https://www.mdpi.com/2227-7390/10/20/3888)
- [Arda — "Bollinger Bands under Varying Market Regimes" (SSRN, 2025)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5775962)
- [Interpretable Hypothesis-Driven Trading: Walk-Forward Validation (arXiv, 2025)](https://arxiv.org/html/2512.12924v1)
- [Assessing Technical Indicators on ML Models (arXiv, 2024)](https://arxiv.org/html/2412.15448v1)
- [Machine Learning Enhanced Multi-Factor Quantitative Trading (arXiv, 2025)](https://arxiv.org/html/2507.07107)
- [Price Pattern Detection Using FSM with Fuzzy Transitions (IEEE, 2014)](https://ieeexplore.ieee.org/document/6982069/)

### Praxis-Blogs und Community-Ressourcen

- [QuantStart — Market Regime Detection with HMM](https://www.quantstart.com/articles/market-regime-detection-using-hidden-markov-models-in-qstrader/)
- [QuantConnect — Intraday Application of HMM](https://www.quantconnect.com/research/17900/intraday-application-of-hidden-markov-models/)
- [QuantInsti — Walk-Forward Optimization](https://blog.quantinsti.com/walk-forward-optimization-introduction/)
- [QuantInsti — Position Sizing Strategies](https://blog.quantinsti.com/position-sizing/)
- [QuantifiedStrategies — ATR Trailing Stops](https://www.quantifiedstrategies.com/average-true-range-trading-strategy/)
- [QuantifiedStrategies — Chandelier Exit](https://www.quantifiedstrategies.com/chandelier-exit-strategy/)
- [StockCharts ChartSchool — Chandelier Exit](https://chartschool.stockcharts.com/table-of-contents/technical-indicators-and-overlays/technical-overlays/chandelier-exit)
- [BuildAlpha — Robustness Testing Guide](https://www.buildalpha.com/robustness-testing-guide/)
- [StrategyQuant — Monte Carlo Methods](https://strategyquant.com/blog/new-robustness-tests-on-the-strategyquant-codebase-5-monte-carlo-methods-to-bulletproof-your-trading-strategies/)
- [Axia Futures — VPOC Reversal Strategy](https://axiafutures.com/blog/volume-profile-vpoc-reversal-strategy/)
- [Volatility Box — Bollinger Bands Squeeze](https://volatilitybox.com/research/bollinger-bands-volatility/)
- [LuxAlgo — ATR Stop Loss Strategies](https://www.luxalgo.com/blog/5-atr-stop-loss-strategies-for-risk-control/)

### Open-Source und Framework-Dokumentation

- [GitHub — Walk-Forward Backtester (Python, Bayesian Optimization)](https://github.com/TonyMa1/walk-forward-backtester)
- [GitHub — Algorithmic Support and Resistance](https://github.com/BatuhanUsluel/Algorithmic-Support-and-Resistance)
- [QuantConnect Walk-Forward Optimization Docs](https://www.quantconnect.com/docs/v2/writing-algorithms/optimization/walk-forward-optimization)

### Weiterführende Literatur (Bücher)

- Van Tharp: "Definitive Guide to Position Sizing" — R-Multiple-Framework
- Alexander Elder: Enthält Chandelier Exit (ursprünglich Charles Le Beau)
- Marcos López de Prado: "Advances in Financial Machine Learning" — CPCV, DSR, Triple Barrier
