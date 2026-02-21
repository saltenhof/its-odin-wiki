# 09 -- Backtesting, Evaluationsdesign, Challenger-Suite, Walk-Forward, Testplan

> **Quellen:** Fachkonzept v1.5 (Kap. 17: Backtesting & Validierungspipeline), Master-Konzept v1.1 (Evaluationsdesign, Challenger Suite), Build-Spec v3.0 (Kap. 13--15: Backtesting, Walk-Forward, Challenger Suite, Bucket-Schwellen; Appendix B: Postgres-Schema; Appendix C: Bucket-Definitionen), Evaluationsdesign (Scenario-/Bucket-Evaluation, Statistische Vergleichsmethodik, Entscheidungslogik), Testplan (Given/When/Then), Strategie-Sparring (Geometric Score, Gate-Kaskade, Indifferenz-Regel, LLM-Kosten-Optimierung)

---

## 1. Ziele (Normativ)

- Realistisches Ausfuehrungsmodell auf OHLCV (kein Lookahead, kein Signal-Bar-Fill)
- Deterministische Replays inklusive LLM ueber CachedAnalyst
- Robustheit gegen Overfitting (Walk-Forward, Sensitivitaet)
- Gleicher Decision-Codepfad in Simulation und Live (Port-Abstraktion)

---

## 2. Governance-Varianten (Ablation-Set)

### 2.1 Primaere Varianten

| Variante | Beschreibung |
|----------|-------------|
| **A: MC-Ansatz (Baseline)** | QuantEngine entscheidet (KPI-first). LLM nur als Tactical Parameter Controller (bounded Enums), keine direkten Trade-Triggers. Entries/Adds strikt Quant-gated |
| **B: Unified-Veto** | QuantEngine generiert Trade-Intents. LLM hat Veto-Recht fuer Entry/Add (`NO_TRADE`/`SAFETY_VETO`), aber kein Recht, Entry/Add zu erzwingen. Fuer Exits darf LLM `EXIT_SOON` triggern, `EXIT_NOW` nur mit Confirm |
| **C: Unified-Full (Dual-Key)** | Entry/Add nur bei QuantVote UND LlmVote (beide >= Confidence-Schwelle, 10m Confirm ok). Exit/Scale-Out durch "one-key pending" + Confirm/Timeout |

### 2.2 Diagnose-Varianten (nicht als produktiver Kandidat)

| Variante | Beschreibung |
|----------|-------------|
| **D: Quant-only** | LLM aus. Misst reinen Mehrwert/Schaden des LLM |
| **E: LLM-only** | Nur als Negativkontrolle. Meistens unterlegen, aber zeigt LLM-Staerken/Schwaechen |

### 2.3 FusionLab-Varianten (isolierte Regime-Fusion-Evaluation)

| Variante | Beschreibung |
|----------|-------------|
| **F0** | MC-Regel (KPI-first, konservativ bei Widerspruch) |
| **F1** | Reliability-weighted (EOD-Update) |
| **F2** | Reliability-weighted (Decision-Level-Update) |
| **F3** | Reliability-weighted + hierarchisches Pooling |

---

## 3. LLM-Determinismus im Backtest (Normativ)

### 3.1 Freeze-Regeln

- **Prompt-Version:** Fixiert fuer den gesamten Run
- **Temperature:** 0 (maximale Reproduzierbarkeit)
- **Top-p:** 1.0 (kein Nucleus-Sampling)
- **Seed:** Fester Wert (konfigurierbar)
- **TTL-Regel:** Identisch in allen Varianten (sonst unfairer "No-Trade"-Bias)

### 3.2 Caching

- Pro `(Symbol, Timestamp, SnapshotHash)` wird der LLM-Output gespeichert
- In allen Varianten wird **derselbe** gecachte Output wiederverwendet
- Cache-Key = `Hash(SystemPromptVersion + SchemaVersion + InputPayloadHash)`
- Im Backtesting MUSS der **CachedAnalyst** verwendet werden

### 3.3 Validity Gate

- JSON-Schema strikt; invalid → zaehlt als "LLM unavailable"
- Loest in allen Varianten denselben Degraded-Mode aus (keine neuen Entries)

### 3.4 Stochastic Audit (Model-Risk)

Einmal pro Setup eine Serie mit `temp=0.2, 10 Replays` — nur um Varianz zu messen, nicht fuer den Hauptvergleich.

---

## 4. Fill-Modell (OHLCV-basiert, konservativ)

### 4.1 Grundregel: Kein Signal-Bar-Fill

Orders DUERFEN im Backtest NICHT auf derselben Bar gefuellt werden, auf der das Signal entsteht. Fills erfolgen fruehestens auf der **naechsten verfuegbaren** 1m-Bar nach Order-Submission.

### 4.2 Limit-Order Fill

- Wenn Limit innerhalb `[Low, High]` der **naechsten** Bar liegt → Fill
- Fill-Preis: konservativ = unguenstigerer Preis innerhalb Bar, alternativ = Limit-Preis
- Wenn nicht erreicht → kein Fill, Repricing/Cancel gemaess OMS-Policy

### 4.3 Stop Fill

- Wenn Stop durch Gap uebersprungen → Fill zum Open der naechsten Bar (worst-case)
- Event `exit_reason = 'STOP_GAP'`

### 4.4 Stress-Fill-Varianten

- Slippage x {1, 2, 3}
- Latenz {0, 1, 2 Minuten}

---

## 5. Kostenmodell (Normativ)

Dasselbe Kostenmodell MUSS in allen Simulationsstufen verwendet werden:

| Kostenkomponente | Modellierung |
|-----------------|-------------|
| **Kommissionen** | IB-Tiered: 0.0035 USD/Share (min. 0.35 USD, max. 1% Handelswert) |
| **Spread-Kosten** | Halber durchschnittlicher Bid-Ask-Spread (Proxy: Funktion von ATR/Preis + Mindest-bps) |
| **Slippage** | Basis: 1 Tick ueber Spread fuer Entries, 0.5 Tick fuer Exits. Bei Volumen < 50.000/Tag: 2 Ticks. Proxy: `slippage = base_slippage + k x ATR x volatility_factor` |
| **Shock Slippage** | Bei Crash/Spike Events: Slippage-Faktor erhoehen |
| **FX-Kosten** | 0.002% (IB-typischer FX-Spread) |
| **Partial Fills** | Nicht modelliert — siehe bekannte Limitationen (Abschnitt 5.1) |

### 5.1 Bekannte Limitation: Partial Fills im Backtest

Partial Fills werden im Backtest nicht modelliert — es wird immer ein vollstaendiger Fill angenommen. Bei den typischerweise gehandelten Instrumenten (populaere High-Beta-Aktien zu regulaeren Handelszeiten) ist dies praktisch irrelevant, da die Liquiditaet fuer die gehandelten Positionsgroessen ausreichend ist.

Die tatsaechliche Auswirkung von Partial Fills wird im **Live-Monitoring** beobachtet (siehe Kapitel 10, Abschnitt 10: Market Surveillance). Keine Backtest-Modellierung fuer V1.

**Abgrenzung:** Im Live-Betrieb werden Partial Fills vollstaendig behandelt (siehe Kapitel 11, Abschnitt 4.8: Partial Fills). Die Limitation betrifft ausschliesslich die Backtest-Simulation.

---

## 6. Datenqualitaet und Daten-Hygiene (Normativ)

> **Quellen:** Unified-Concept v2.1 (12.2 Daten-Hygiene), Build-Spec v3.0 (49.4 Corporate Actions Handling)

### 6.1 Corporate Actions

- **Splits und Reverse-Splits:** OHLCV-Historien MUESSEN split-adjustiert sein (Preis UND Volumen). Nicht-adjustierte Daten fuehren zu falschen ATR-Werten, falschen Breakout-Signalen und fehlerhaftem Position Sizing.
- **Dividenden:** Fuer Intraday-Backtesting auf Tagesbasis vernachlaessigbar (kein Overnight-Holding). Ex-Dividend-Gaps am Tagesopen werden durch das Fill-Modell (kein Signal-Bar-Fill) abgefangen.
- **Konsistenz:** Die Adjustierung MUSS zwischen Backtest und Live identisch sein. Wenn der Datenanbieter adjustiert, muss dieselbe Adjustierungsmethode dokumentiert und fixiert werden.

### 6.2 Survivorship Bias

- Backtesting-Universen SOLLEN delistete und uebernommene Aktien enthalten (wenn verfuegbar), um Survivorship Bias zu minimieren.
- Wenn nur ueberlebende Symbole verfuegbar sind: Dokumentieren und als bekannte Limitation im Run-Report ausweisen (Feld `notes` in `bt_run`).

### 6.3 Timezone und Kalender

- Alle Timestamps in **US/Eastern** (Exchange-Time) oder **UTC** mit dokumentiertem Offset
- Handelstage: NYSE/NASDAQ-Kalender (keine Wochenenden, Feiertage, fruehe Schliessungen beruecksichtigen)
- Halts: Fehlende Bars innerhalb RTH werden durch DQ-Gates erkannt (siehe Challenger S13)

---

## 7. Datenstufen im Backtest (Normativ)

### 7.1 Datenstufen-Tagging

Jeder Backtest-Run wird mit seiner Datenstufe getaggt. Die Stufe bestimmt, welche Marktdaten dem Backtest zur Verfuegung stehen:

| Datenstufe | Beschreibung | Verfuegbarkeit |
|------------|-------------|----------------|
| **OHLCV_ONLY** | Standard. Nur OHLCV-Bars (1m/3m/10m). Keine Bid/Ask-Spreads, kein Level-2 | Immer verfuegbar (historische Daten) |
| **BASIC_QUOTES** | Mit aufgezeichneten Bid/Ask-Daten. Ermoeglicht realistischere Spread-Modellierung | Nur wenn im Live-Betrieb aufgezeichnet |
| **L2_DEPTH** | Mit aufgezeichneten Level-2-Orderbuchdaten. Ermoeglicht Liquiditaetsanalyse und praezisere Fill-Modellierung | Nur wenn im Live-Betrieb aufgezeichnet |

### 7.2 Zentrale Regel: Datenkonsistenz

**Alpha-Signale duerfen nur auf Daten basieren, die auch im Backtest desselben Laufs verfuegbar waren.** Ein Signal, das auf Bid/Ask-Spreads basiert, darf nicht in einem OHLCV_ONLY-Backtest verwendet werden — auch wenn es im Live-Betrieb verfuegbar waere.

Backtest-Ergebnisse verschiedener Datenstufen duerfen **nicht direkt verglichen** werden. Sie muessen als separate Runs mit explizitem Datenstufen-Tag gefuehrt werden, da unterschiedliche Datengrundlagen die Ergebnisse strukturell beeinflussen.

### 7.3 Recording-Modus (Live → Backtest)

Wenn Bid/Ask- oder L2-Daten im Live-Betrieb verfuegbar sind, SOLLEN diese aufgezeichnet werden, um spaetere Backtests mit reicheren Daten fuettern zu koennen. Dieses Recording baut ueber Zeit eine zunehmend detailliertere Datenbasis auf und ermoeglicht schrittweise den Uebergang von OHLCV_ONLY zu BASIC_QUOTES und spaeter L2_DEPTH.

Die Recording-Konfiguration wird pro Instrument festgelegt und im Run-Metadaten-Feld `global_config` dokumentiert.

---

## 8. Bewertungsmetriken (geometrisch priorisiert)

### 8.1 Primaermetriken (entscheidend)

| Metrik | Formel/Definition |
|--------|------------------|
| **Geometric Mean Daily Return** | `G = exp(mean(ln(1 + r_d))) - 1` |
| **Max Drawdown (MDD)** | Maximaler Peak-to-Trough-Drawdown auf Equity |
| **Expected Shortfall (CVaR 95%)** | Durchschnitt der schlechtesten 5% Daily-Log-Returns |
| **Tail-Loss Frequency** | Anteil Tage mit Return < -X% (z.B. -2 Sigma oder feste Schwelle) |
| **Recovery Time** | Dauer vom Drawdown-Start bis zum Recovery auf vorheriges Peak |

### 8.2 Sekundaermetriken

- Profit Factor, Hit Rate, Payoff Ratio
- MAE/MFE pro Trade, Giveback (MFE vs. realized)
- Trades/Tag, Order Churn, Cancel/Replace Rate
- Anteil "No-Trade Days" (zu restriktiv kann Opportunity zerstoeren)
- "Capture Ratio": Anteil an Tagesrange eingefangen
- "Overtrade Score": Unnoetige Trades in Chop
- "Safety Score": Verhalten in Crash/Halt Szenarien

### 8.3 Geometric Score (Ranking-Funktion)

```
S = mean(ln(1 + r_d)) - lambda x |ES_95(ln(1 + r_d))| - mu x |MDD|
```

**Startwerte (Stakeholder-Entscheidung):**

- `lambda = 1.0`
- `mu = 2.0` (MDD staerker bestrafen als ES95, weil Drawdown geometrisch dominiert)

**Kalibrierpfad (verhindert Overfit):**

- Grid nur auf Train-Fenstern: `lambda in {0.5, 1.0, 1.5, 2.0}`, `mu in {1.0, 2.0, 3.0}`
- Auswahlregel: nicht "Bestpunkt", sondern **Plateau** (robust gegen +/- 10%)

---

## 9. Scenario-Buckets (OHLCV-only, konfigurierbar)

Jeder Handelstag wird in Buckets klassifiziert. Bucket-Labeling erfolgt deterministisch aus OHLCV-Daten.

### 9.1 Opening Shock Down

- Innerhalb der ersten 10 Minuten: `drawdown_from_open <= -4%` ODER (`<= -3%` UND `vol_ratio >= 3.0`)

### 9.2 Parabolic + Plateau + Fade

1. **Run-up frueh:** `time_of_day(high_of_day) <= open + 120min` UND (`run = (HOD - Open) / Open >= 0.04` ODER `run >= 0.60 x ADR14`)
2. **Plateau:** Ab `t_high` mindestens 30 Minuten, in denen `rolling_range_30m <= 0.8 x ATR10m(t_high)` (seitwaerts nahe Hoch)
3. **Fade / Giveback:** `giveback = (HOD - Close) / (HOD - Open) >= 0.60` UND letzter 60-Min-Return negativ: `Close - Close(t_close-60m) <= -0.5 x ATR10m`

### 9.3 Compression → Breakout

1. **Compression-Phase (mind. 60 Min):** `compression_range / baseline_range <= 0.35` UND `ATR10m_slope_60m < 0`
2. **Breakout-Impuls (innerhalb 30 Min):** `breakout_move >= 1.5 x ATR10m` UND `vol_ratio_10m >= 1.8`

### 9.4 VWAP Whipsaw (Chop um VWAP)

- `cross_count_120m >= 8` (Signwechsel von close-vwap auf 3m Bars)
- `ADX10m <= 15`
- `abs(Close - Open) <= 0.25 x ADR14`

### 9.5 Trend Day Grind Up

Monotoner Aufwaertstrend ueber den gesamten Handelstag mit wenig Pullbacks. Dieser Bucket repraesentiert den idealen Trend-Day-Fall, in dem das System den Trend reiten und den Runner moeglichst lange halten SOLL.

**OHLCV-Kriterien (deterministisch):**

- **Directional Consistency:** `close > open` fuer >= 80% der 3m-Bars waehrend RTH (Mindestens 4 von 5 Bars bullish)
- **Trend-Staerke:** `ADX(14, 10m) > 25` ueber mindestens 80% der 10m-Bars waehrend RTH
- **Higher-Highs / Higher-Lows:** Auf 10m-Ebene monotone Folge von Higher-Highs UND Higher-Lows (maximal 1 Violation erlaubt)
- **EMA-Alignment:** `EMA(9) > EMA(21)` auf 10m-Ebene durchgehend (keine Kreuzung waehrend RTH)
- **Close nahe HOD:** `Close >= HOD - 0.3 x ATR(14, 3m)` (Tagesschluss nahe am Tageshoch, kein spaeter Giveback)
- **Kein spaeter Dump:** Letzte 60 Minuten Return >= -0.25 x ATR(14, 10m) (kein signifikanter Ruecksetzer zum Schluss)

**Abgrenzung zu anderen Buckets:**

- Unterschied zu Parabolic+Plateau+Fade: Kein fruehes HOD, kein Plateau, kein Giveback
- Unterschied zu Compression→Breakout: Kein Kompressionsmuster, sondern stetiger Anstieg von Anfang an

### 9.6 Halt/Resume Proxy

- Fehlende Bars / grosse Gaps innerhalb der Session
- Keine Updates > 5 Minuten waehrend RTH

### 9.7 Bucket-Report

Fuer jeden Bucket wird separat reported:

- Geometric Mean Log-Return
- Tail-Frequency
- MDD und Recovery Time
- Overtrade Score

---

## 10. Walk-Forward Validation (Normativ)

### 10.1 Splits

- **Trainingsperiode:** 60 Handelstage
- **Validierungsperiode:** 20 Handelstage
- **Schrittweite:** 20 Tage (Rolling Window)
- **Kriterium:** Parameter MUESSEN in mindestens **3 von 4** aufeinanderfolgenden Validierungsperioden profitabel sein UND Risk-Metriken innerhalb Limits liegen

### 10.2 Was wird kalibriert?

- Nur Schwellen/Parameter, die **alle Varianten teilen** (z.B. Crash-Thresholds, ATR-Decay)
- LLM-Arbitrations-Schwellen (`min_conf`) duerfen variieren, aber MUESSEN per Walk-Forward auf Out-of-Sample optimiert werden

### 10.3 Keine Varianten-Optimierung im Test

Parameter werden pro Fold nur auf dem Train-Fenster gewaehlt und dann fix auf Test angewandt.

---

## 11. Stress-Tests (Normativ)

Fuer jede Variante zusaetzlich:

| Test | Beschreibung |
|------|-------------|
| **Slippage x2/x3** | Erhoehte Transaktionskosten |
| **Latenz 1--2 Minuten** | Verzoegerte Ausfuehrung |
| **Bar-Alignment Shift** | 3m/10m Aggregation um 1 Minute versetzt |
| **Cooldown haerter/weicher** | Overtrade-Sensitivitaet |
| **LLM Outage Simulation** | 5%/10% random missing Outputs → Degraded Mode |

**Erfolgskriterium:** Score und Tail-Frequenz DUERFEN NICHT kollabieren.

---

## 12. Ablation Tests (Normativ)

MUSS mindestens auswerten:

- **Quant-only** (LLM aus)
- **LLM-only** (nur als Experiment, nicht produktiv)
- **Hybrid** (alle Varianten A/B/C)

Ziel: Mehrwert des LLM objektiv messen.

---

## 13. Statistische Vergleichsmethodik

### 13.1 Paired Day Comparison

- Gleiche Tage, gleiche Symbole → Differenz der Daily-Log-Returns pro Tag
- Reporte: Mittel/Median der Differenzen, 5/50/95-Perzentile

### 13.2 Bootstrap

- Block Bootstrap ueber Tage (um Autokorrelation zu respektieren)
- Wahrscheinlichkeit, dass Unified > MC nach Score (Posterior/Bootstrap)
- Nicht nur Signifikanz, sondern **Effektgroesse + Tail-Verhalten**

### 13.3 Power-Abschaetzung

Fuer Paired Day Test (`alpha=5%, Power 80%`):

| Stddev der Tages-Differenzen | Effekt 10 bps | Effekt 5 bps |
|------------------------------|--------------|-------------|
| 0.6% | ~282 Handelstage | ~1129 Handelstage |
| 1.0% | ~784 Handelstage | Unpraktikabel |

**Konsequenz:** Kleine P&L-Verbesserungen sind schwer sauber zu beweisen. FusionLab (viele 3m-Entscheidungen pro Tag) liefert statistisch schneller Ergebnisse.

---

## 14. Entscheidungslogik: Gate-Kaskade (Stakeholder-Entscheidung)

### Gate 0 -- Safety (hart, geometrisch)

Unified wird **abgelehnt**, wenn es gegenueber MC **Tail-Risiko** klar verschlechtert:

- `MDD_unified > MDD_mc x 1.10` → FAIL
- `ES95_unified < ES95_mc - 0.0005` (log-return ES, deutlich schlechter) → FAIL
- `TailLossFreq_unified > TailLossFreq_mc + 20% relativ` → FAIL

### Gate 1 -- P&L Signifikanz (wenn vorhanden)

- Wenn Score Unified > MC **signifikant** UND Gate 0 bestanden → Unified gewinnt

### Gate 2 -- Indifferenz-Fall (keine Signifikanz)

Wenn keine statistische Signifikanz, dann in Reihenfolge:

1. **FusionLab Proxy:** Wenn F2/F3 Regime-Accuracy und/oder Kostenfunktion deutlich verbessert UND Gate 0 bestanden → Unified-Veto gewinnt
2. **Bucket-/Stress-Robustheit:** Wenn Unified in Crash/Whipsaw/Plateau-Fade Buckets nicht schlechter und in 1--2 Kern-Buckets klar besser → Unified-Veto gewinnt
3. **Default-Regel:** Bei Indifferenz gilt **MC** (konservativ, sofort einsatzfaehig)

> **Warum Unified-Veto als Default-Upgrade:** Bei Indifferenz ist Unified-Full das hoehere Tail-Risiko (mehr Hebel fuer LLM). Unified-Veto liefert typischerweise den Nutzen "Bad Trades vermeiden" bei deutlich geringerem zusaetzlichen Risiko.

---

## 15. Release-Gate Pipeline (Normativ)

| Stufe | Modus | Dauer | Kriterium |
|-------|-------|-------|-----------|
| **1. Backtest Suite** | Historische Daten | Gesamte verfuegbare Historie | Challenger Suite bestanden, Walk-Forward profitabel |
| **2. Paper Trading** | Simulated Orders gegen echte Marktdaten (IB-Paper-Account) | Min. 20 Handelstage | Simulated P&L positiv, Drawdown < 10%, Win-Rate > 40% |
| **3. Klein-Live** | Echtes Trading mit 10% des Normal-Kapitals (1.000 EUR) | Min. 20 Handelstage | P&L positiv, keine unerwarteten Failure Modes |
| **4. Live** | Volles Kapital | Laufend | Monitoring, Alert-System aktiv |

Jede Stufe hat **Abbruchkriterien** (Drawdown, Incident-Rate, Unexpected Failure Modes).

---

## 16. Challenger-Suite: 20 Intraday-Szenarien (Normativ)

### 16.1 Testformat

Jeder Challenger definiert:

- **Input:** 1m OHLCV Serie (ggf. Daily-Kontext)
- **Erwartete Detektion:** Welche MonitorEvents/Flags MUESSEN entstehen
- **Erwartete erlaubte Aktionen:** Entry/Exit/Scale/Block
- **Erwartete Safety-Reaktionen:** Stop-Updates, Degradation, EOD flat
- **Pass/Fail-Kriterien:** Deterministische Bedingungen (nicht nur P&L)

### 16.2 Die 20 Szenarien

| # | Szenario | Detection (Kurz) | Erwartetes Verhalten |
|--:|----------|-----------------|---------------------|
| **S01** | Opening Spike → Konsolidierung → Trend up | ATR-Decay, Contraction, Breakout + Vol | Setup A Entry nach Buffer, Add nach HL. Profit Protection ab 1R |
| **S02** | Opening Spike → Fake Breakout → Range Day | Spike, kurzer Ausbruch, zurueck zur VWAP | LLM/Quant RedFlag "fakeout/chop". Entry blockiert oder schnell beendet. Overtrade-Schutz |
| **S03** | -10% Flush → V-Reversal → Rally | FLUSH_DETECTED + Vol Spike | Starter-Entry erst nach RECLAIM. Add erst nach 10m Confirmation/HL |
| **S04** | -10% Flush → Dead-Cat-Bounce → weiter DOWN | FLUSH erkannt, aber Reclaim-Gate nicht erfuellt | Setup wird abgebrochen. Kein "Averaging down" |
| **S05** | Coil ueber 2h → Breakout midday | COIL_FORMING, Range-Kompression | Entry Breakout oder Retest. Parabolic-Protection aktiv wenn Ausbruch explodiert |
| **S06** | Coil → Fakeout → Stop small loss | Breakout Entry, sofortige Negation | Hard-Stop greift schnell. Cooldown verhindert sofortigen Re-Entry |
| **S07** | Parabolic Run → Plateau → Dump into Close | Parabolic/Exhaustion Events | Scale-Out-Ladder vor Dump. EOD Forced Close zuverlaessig |
| **S08** | Slow Trend Day (steady up) | 10m Trend stabil, wenige Pullbacks | Kein hektisches Scalping. Runner bleibt, Trailing folgt. Nicht zu frueh alles verkaufen |
| **S09** | VWAP Magnet Chop Day | ADX low, ATR-Decay, haeufige VWAP-Crosses | RANGE Regime. Entry-Gates restriktiv. Max-Trades und Cooldown verhindern Overtrade |
| **S10** | Gap Up & Fade | Gap up, Abverkauf unter VWAP | Entry nur nach Reclaim. Schneller Exit wenn VWAP nicht haelt |
| **S11** | Gap Down & Recover | Gap down, Erholung | Keine Panik-Entries; nur nach Reclaim + Confirmation |
| **S12** | News-aehnlicher Shock Dump | Crash Event intraday | Stop/Exit ohne LLM. Degradation moeglich |
| **S13** | Volatility Halt / Datenluecke | Ploetzlich fehlende Bars | DQ_STALE_FEED, Trading Halt. Keine neuen Orders, nur Risk-reducing |
| **S14** | Datafeed Outlier-Bar (Fehler) | Absurd hohe Range, kein Volume-Spike | Outlier Filter verwirft Bar. Keine falschen Trades |
| **S15** | LLM Outage mitten im Trade | Timeouts / invalid Schema | QUANT_ONLY Mode. Keine neuen Entries. Bestehende Position deterministisch managen |
| **S16** | Broker Reject bei Order | Order wird abgelehnt | CRITICAL Alert. Retry begrenzt. Ggf. Kill-Switch |
| **S17** | Partial Fills (simuliert) | Teilfuellung | Position/Stops/Targets korrekt skaliert. Keine doppelte Exposure |
| **S18** | Re-Entry nach Take Profit + Pullback + neuer Trend | Setup D | Cycle2 erlaubt nach Cooldown. Sizing reduziert. Profit Protection frueher |
| **S19** | Kein Entry vormittags (DOWN), Trendwechsel nachmittags | Setup D aktiv | 10m Confirmation verhindert zu fruehes "Catch the falling knife" |
| **S20** | End-of-Day Squeeze → Reversal | Late Ramp + Reversal | Neue Entries nach Cutoff blockiert. Offene Position geschuetzt/geschlossen (EOD flat) |

### 16.3 Regression Report

Jeder Challenger erzeugt:

- Pass/Fail je Erwartung
- Timeline der Events/States
- Decisions + ReasonCodes
- Optional: Chart-Overlay aus EventLog

---

## 17. Testplan (Given/When/Then) mit Postgres-Backend

### 17.1 Testgruppen

1. Data Integrity & Resampling (T1--T3)
2. Determinismus & Replay inkl. LLM Cache (T4--T6)
3. Execution/Fill/Kostenmodell (T7--T10)
4. Risk/Guardrails (T11--T13)
5. Governance-Logik MC vs Unified (T14--T17)
6. Challenger Suite (20 Szenarien)
7. Geometrische KPI-Auswertung (SQL-Checks)

### 17.2 Kerntests

| Test | Given | When | Then |
|------|-------|------|------|
| **T1: Dataset Checksum** | Importiertes OHLCV-1m Dataset | `checksum_sha256` berechnet | Jeder Run referenziert selben Checksum, sonst Run abort |
| **T2: Bar-Vollstaendigkeit** | 1m Bars pro Symbol/Tag | Bars validiert | Keine fehlenden Minuten in RTH, keine Duplikate, keine OHLC-Inkonsistenzen |
| **T3: Resampling** | 1m Sequenzen | 3m/10m resampled | O=Open(1.), C=Close(last), H=max, L=min, V=sum |
| **T4: Determinismus ohne LLM** | Quant-only mit fixem Seed | Run 2x ausgefuehrt | `bt_day` und `bt_trade` bitgleich |
| **T5: LLM Cache Replay** | LLM-Variante mit snapshot_hash | Run erneut ausgefuehrt | Cache wiederverwendet, `bt_day` identisch |
| **T6: LLM Schema Validation** | LLM Output verletzt Schema | Gespeichert | `valid=false`, Variante verhaelt sich gemaess Degraded-Policy |
| **T7: Next-Bar Fill** | Entry-Intent am 3m Decision Close | Order simuliert | Fill fruehestens naechste 1m-Bar |
| **T8: Limit Fill** | Limitpreis L | Naechste Bar `low<=L<=high` | Fill zum Limit oder konservativer |
| **T9: Stop Fill + Gap** | StopTrigger S, Bar Open < S | Fill simuliert | Fill zum Open, `exit_reason='STOP_GAP'` |
| **T10: Slippage/Latency Stress** | Identische Signale | `slippage_mode=x2/x3` | PnL sinkt, Tail-Risk steigt (Sanity) |
| **T11: EOD Flat** | Offene Positionen vor Close | Forced Close Window | Alle geschlossen, `exit_reason='FORCED_CLOSE'` |
| **T12: Max Daily DD** | DD-Limit ueberschritten | -- | `DAY_STOPPED`, keine neuen Trades |
| **T13: Max Cycles** | `max_cycles=3`, Counter==3 | -- | `BLOCK_REENTRY`, keine Entries |
| **T14: MC Governance** | MC-Variante, LLM sagt STRONG_ENTRY, Quant sagt NO_TRADE | -- | Kein Entry |
| **T15: Unified-Veto** | Quant ALLOW_ENTRY, LLM SAFETY_VETO | -- | Kein Entry, `reject_reason` geloggt |
| **T16: Unified-Full Dual-Key** | Nur eine Seite ALLOW_ENTRY | -- | Kein Entry |
| **T17: Exit-Pending + Confirm** | LLM EXIT_NOW ohne Confirm | -- | PENDING_EXIT, Exit nur bei Confirm/Timeout |

---

## 18. Datenbank-Schema (Normativ)

### 18.1 Kernschema (9 Tabellen)

```sql
-- 1) Datensatz-Registry
create table if not exists bt_dataset (
  dataset_id bigserial primary key,
  name text not null,
  source text not null,
  start_date date not null,
  end_date date not null,
  ohlcv_granularity text not null default '1m',
  checksum_sha256 text not null,
  created_at timestamptz not null default now()
);

-- 2) Run-Metadaten
create table if not exists bt_run (
  run_id bigserial primary key,
  dataset_id bigint not null references bt_dataset(dataset_id),
  engine_version text not null,
  data_tier text not null default 'OHLCV_ONLY', -- OHLCV_ONLY | BASIC_QUOTES | L2_DEPTH
  created_at timestamptz not null default now(),
  notes text,
  global_config jsonb not null
);

-- 3) Varianten
create table if not exists bt_variant (
  variant_id bigserial primary key,
  run_id bigint not null references bt_run(run_id) on delete cascade,
  variant_name text not null,
  governance text not null,
  prompt_version text,
  model_version text,
  random_seed int not null default 0,
  latency_mode text not null default '0m',
  slippage_mode text not null default 'base',
  bar_alignment_shift_minutes int not null default 0,
  variant_config jsonb not null,
  created_at timestamptz not null default now()
);

-- 4) Daily-Metriken
create table if not exists bt_day (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  trading_date date not null,
  equity_start numeric(20,8) not null,
  equity_end numeric(20,8) not null,
  daily_return numeric(18,10) not null,
  log_return numeric(18,10) not null,
  gross_pnl numeric(20,8) not null,
  net_pnl numeric(20,8) not null,
  fees numeric(20,8) not null,
  slippage_cost numeric(20,8) not null,
  trades_count int not null,
  orders_count int not null,
  churn_score numeric(18,10) not null,
  primary key (variant_id, trading_date)
);

-- 5) Trades
create table if not exists bt_trade (
  trade_id bigserial primary key,
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  symbol text not null,
  entry_time timestamptz not null,
  exit_time timestamptz not null,
  qty numeric(20,8) not null,
  entry_price numeric(20,8) not null,
  exit_price numeric(20,8) not null,
  initial_stop numeric(20,8),
  pnl_net numeric(20,8) not null,
  r_multiple numeric(18,10),
  mfe numeric(20,8),
  mae numeric(20,8),
  cycle_no int not null,
  entry_reason text,
  exit_reason text,
  tags jsonb not null default '{}'::jsonb
);

-- 6) Decision Cycles (3m)
create table if not exists bt_decision_cycle (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  symbol text not null,
  decision_time timestamptz not null,
  state text not null,
  regime_final text not null,
  intent text not null,
  reject_reason text,
  quant_vote text,
  quant_conf numeric(6,5),
  llm_vote text,
  llm_conf numeric(6,5),
  confirm_10m text,
  snapshot_hash text not null,
  primary key (variant_id, symbol, decision_time)
);

-- 7) Monitor Events (1m)
create table if not exists bt_monitor_event (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  symbol text not null,
  event_time timestamptz not null,
  event_type text not null,
  severity int not null default 1,
  payload jsonb not null default '{}'::jsonb,
  primary key (variant_id, symbol, event_time, event_type)
);

-- 8) LLM Cache
create table if not exists bt_llm_cache (
  snapshot_hash text primary key,
  prompt_version text not null,
  model_version text not null,
  created_at timestamptz not null default now(),
  valid boolean not null,
  output jsonb,
  error text,
  latency_ms int
);

-- 9) Szenario-/Bucket-Labels
create table if not exists bt_day_bucket (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  trading_date date not null,
  symbol text not null,
  bucket text not null,
  bucket_score numeric(18,10),
  details jsonb not null default '{}'::jsonb,
  primary key (variant_id, trading_date, symbol, bucket)
);
```

### 18.2 FusionLab-Tabellen

Siehe Kapitel 08 (Symmetric Hybrid Protocol) fuer `bt_ground_truth` und `bt_regime_eval`.

---

## 19. SQL-Auswertungen (Referenz)

### 19.1 Geometric Mean Daily Return

```sql
select v.variant_id, min(v.variant_name) as variant,
       exp(avg(d.log_return)) - 1 as geo_mean_daily_return
from bt_day d join bt_variant v using (variant_id)
group by v.variant_id
order by geo_mean_daily_return desc;
```

### 19.2 Geometric Score S

```sql
with base as (
  select variant_id, avg(log_return) as avg_log from bt_day group by variant_id
),
mdd as (
  with eq as (
    select variant_id, trading_date, equity_end,
           max(equity_end) over (partition by variant_id order by trading_date) as peak
    from bt_day
  )
  select variant_id, min(equity_end/peak - 1) as max_dd from eq group by variant_id
),
es as (
  with r as (select variant_id, log_return from bt_day),
  p as (
    select variant_id,
           percentile_cont(0.05) within group (order by log_return) as p05
    from r group by variant_id
  )
  select r.variant_id, avg(r.log_return) filter (where r.log_return <= p.p05) as es95
  from r join p using (variant_id) group by r.variant_id
)
select v.variant_id, min(v.variant_name) as variant,
       b.avg_log, es.es95, mdd.max_dd,
       (b.avg_log - (1.0 * abs(es.es95)) - (2.0 * abs(mdd.max_dd))) as score_s
from base b
join es using (variant_id) join mdd using (variant_id)
join bt_variant v on v.variant_id = b.variant_id
group by v.variant_id, b.avg_log, es.es95, mdd.max_dd
order by score_s desc;
```

### 19.3 Paired Day Comparison (MC vs Unified)

```sql
with mc as (
  select trading_date, log_return from bt_day d
  join bt_variant v using (variant_id) where v.variant_name = 'MC'
),
uv as (
  select trading_date, log_return from bt_day d
  join bt_variant v using (variant_id) where v.variant_name = 'UNIFIED_VETO'
)
select mc.trading_date, (uv.log_return - mc.log_return) as delta_log_return
from mc join uv using (trading_date)
order by mc.trading_date;
```

---

## 20. LLM-Kosten-Optimierung (stufenweise)

### Phase 0 -- Ohne LLM (billig, schnell)

- Quant-only auf maximal verfuegbarem Zeitraum
- Ziel: Bucket-Labeler stabilisieren, Kostenmodell stabilisieren, Baseline-Volatilitaet schaetzen

### Phase 1 -- LLM nur auf Subset + Cache (einmal zahlen, oft nutzen)

- LLM-Calls eventgetrieben / im festen Takt, **einmalig** in Cache persistieren
- Alle Governance-Varianten (MC/Unified-Veto/Unified-Full + FusionLab F0--F3) laufen im Replay ueber den **gleichen Cache** → Kosten steigen NICHT linear mit Varianten

### Phase 2 -- Walk-Forward-Umfang realistisch ableiten

- Minimum fuer 4 Validierungsfenster: ~140 Handelstage
- FusionLab + Bucket-Stabilitaet + Stress-Tests als zusaetzliche Evidenz
