# 01 -- Datenpipeline, Timeframes, Buffers, DQ-Gates, Feature-Katalog

> **Quellen:** Fachkonzept v1.5 (Kap. 12: Datenarchitektur & Subscriptions), Master-Konzept v1.1 (Data Pipeline, Buffer, DQ Gates), Build-Spec v3.0 (Kap. 3: Daten-Layer, Feature-Katalog), Strategie-Sparring (OHLCV-Constraint, L2-Kompromiss, Timeframe-Hierarchie), Stakeholder-Feedback (1m/3m/10m Entscheidung)

---

## 1. Datenverfuegbarkeit und -restriktion

### 1.1 Konfigurationsabhaengige Datenstufen

Die Datenverfuegbarkeit ist NICHT statisch, sondern konfigurationsabhaengig ueber `odin.data.subscriptions`. ODIN MUSS in jeder Stufe funktionsfaehig sein:

| Stufe | Inhalt | Typischer Einsatz |
|-------|--------|-------------------|
| **OHLCV_ONLY** | Nur OHLCV-Bars | Backtest (Standard), Live-Minimalbetrieb |
| **BASIC_QUOTES** | OHLCV + Bid/Ask | Live mit echter Spread-Pruefung, bessere Execution |
| **L2_DEPTH** | OHLCV + Bid/Ask + Level-2-Orderbuch | Beste Execution-Qualitaet im Live-Betrieb |

**Zentrale Designregel:** Features, die optional sind, haben immer einen Proxy-Fallback. Alpha-Signale DUERFEN nur auf Daten basieren, die auch im Backtest desselben Laufs verfuegbar waren.

**Harter Satz:** Bid/Ask und L2-Daten DUERFEN live fuer Execution und Risk genutzt werden, aber NIE als Alpha-Feature -- es sei denn, der Backtest-Run basiert auf derselben Datenstufe.

**Konsequenzen auf OHLCV_ONLY-Stufe:**

- Alpha-Generierung, Regime-Erkennung und Setup-Aktivierung basieren ausschliesslich auf OHLCV
- Liquiditaets- und Ausfuehrungsqualitaet wird ueber Proxies modelliert (Volumen/Range/Fill-Rate)
- Alle Signale MUESSEN aus OHLCV ableitbar sein
- Backtest: konservatives Kosten-/Fill-Modell + Stress-Szenarien

**Recording-Modus:** Wenn Bid/Ask oder L2 im Live-Betrieb verfuegbar sind, KOENNEN diese Daten aufgezeichnet werden, um spaetere Backtests auf derselben Datenstufe zu ermoeglichen. Backtest-Runs werden mit ihrer Datenstufe getaggt.

### 1.2 Datenverfuegbarkeits-Matrix

Die folgende Matrix zeigt, welche Daten in welchem Modus verfuegbar sind:

| Daten | Backtest (Standard) | Backtest (mit Recording) | Live (OHLCV_ONLY) | Live (BASIC_QUOTES) | Live (L2_DEPTH) |
|-------|--------------------|-----------------------|-------------------|-------------------|----------------|
| OHLCV 1m/3m/5m | Ja | Ja | Ja | Ja | Ja |
| Bid/Ask | Nein | Ja (recorded) | Nein (Proxy) | Ja | Ja |
| L2 Orderbook | Nein | Ja (recorded) | Nein | Nein | Ja |
| News/Headlines | Nein | Nein | Optional | Optional | Optional |

**Proxy-Fallbacks bei fehlenden Daten:**

- **Bid/Ask nicht verfuegbar:** Spread-Schaetzung ueber Volumen und Volatilitaet (konservativ). Fill-Modell mit erhoehtem Slippage.
- **L2 nicht verfuegbar:** Liquiditaets-Assessment ueber Volumen-Ratio und historische Fill-Raten.
- **News nicht verfuegbar:** Keine Einschraenkung der Kernfunktionalitaet. News sind stets optional und nie Alpha-kritisch.

---

## 2. Drei-Layer-Timeframe-Architektur (Stakeholder-Entscheidung 2026-02-26)

| Layer | Timeframe | Quelle | Rolle | Lookback-Minimum |
|-------|-----------|--------|-------|------------------|
| **Event-Detektor** | **1m** | Direkt vom Feed | NUR Monitoring fuer extreme/ueberproportionale Moves (+/-6% Spike). KEIN normaler Decision-Layer. Stop-Tightening, Crash/Spike-Detection | -- |
| **Decision-Layer** | **3m oder 5m** (parametrisierbar) | Resampled aus 1m | Hauptlage-Beurteilung: Entry/Exit/Scale-Entscheidungen, EMA(9), EMA(21) auf Decision-Bars | EMA(21): mind. 22 Bars |
| **Fixe KPI-Timeframe** | **immer 5m** | Resampled aus 1m | ATR(14), ADX(14), RSI(14), Bollinger(20,2) — IMMER auf 5m, unabhaengig vom Decision-Layer | ATR(14), RSI(14), ADX(14): mind. 15 Bars; Bollinger(20): mind. 21 Bars |
| **Daily** | -- | Aus Historie | ADR(14), Gap, Prior Day Levels | 30 Tage |

**Wichtig:** Der Decision-Layer-Timeframe ist parametrisierbar via `odin.data.decision-bar-timeframe-s` (Default: 180 fuer 3m, alternativ 300 fuer 5m). Praeferenz fuer High-Beta-Aktien: 3m. Standard: 5m. ATR, ADX, RSI werden IMMER auf 5m berechnet — auch wenn der Decision-Layer auf 3m laeuft.

**10m-Confirmation-Loop entfaellt.** Die Regime-Hysterese wird durch Confirmation-Lag auf Decision-Bars geloest (2 aufeinanderfolgende Decision-Bars mit konsistentem neuem Regime).

---

## 3. Resampling-Regeln (Normativ)

Resampling erfolgt **zeitbasiert und bar-close-basiert** in einer Aggregationskaskade: 1m -> 3m -> 5m.

**3m-Bar** = Aggregation aus drei 1m-Bars:

- O = Open(Bar1)
- H = max(High aller 3 Bars)
- L = min(Low aller 3 Bars)
- C = Close(Bar3)
- V = Sum(Volume aller 3 Bars)

**5m-Bar** = Aggregation aus fuenf 1m-Bars (analog). Wird IMMER erzeugt, da KPIs (ATR, ADX, RSI) fix auf 5m laufen.

**Alignment:** Resampling ist an Session-Start ausgerichtet (z.B. 3m: 09:30:00 -> 09:32:59, 09:33:00 -> 09:35:59; 5m: 09:30:00 -> 09:34:59, 09:35:00 -> 09:39:59).

**Gueltigkeitsregel:** Eine aggregierte Bar ist gueltig erst nach Close der letzten enthaltenen 1m-Bar. Keine vorzeitige Auswertung.

### 3.1 Incomplete-Bar-Policy (Normativ, harte Anforderung)

Bei einem erzwungenen 1m-Event-Interrupt (z.B. durch ein CRITICAL-Event wie CRASH_DOWN oder EXHAUSTION_CLIMAX) friert die Pipeline den State der letzten vollstaendig geschlossenen Decision-Bars und 5m-Bars ein.

**Regel:** Berechnungen basieren ausschliesslich auf dem Snapshot der letzten abgeschlossenen Bars (T-1) plus den dedizierten 1m-Features der aktuellen, triggernden 1m-Bar. Es findet KEINE Neuberechnung von gleitenden Durchschnitten, Volatilitaetsbaendern oder sonstigen Multi-Bar-Indikatoren auf Basis unfertiger Bars statt.

**Begruendung:** Diese Regel ist eine harte Anforderung fuer Backtest-Fidelity. Wuerde man unfertige Bars in KPI-Berechnungen einbeziehen, waeren die Ergebnisse nicht reproduzierbar -- die gleiche Marktsituation koennte je nach Zeitpunkt des Interrupts zu unterschiedlichen Indikatorwerten fuehren.

**Konsequenzen:**

- 1m-Features (Jump/Crash Detection, Wick Ratios, Micro-Structure) werden normal auf der aktuellen 1m-Bar berechnet
- Decision-Bar-KPIs (EMA) und 5m-KPIs (RSI, ADX, ATR, Bollinger) verwenden den letzten abgeschlossenen Bar des jeweiligen Timeframes
- VWAP wird kumulativ berechnet und ist davon nicht betroffen (jeder 1m-Bar-Close aktualisiert VWAP)

---

## 4. Rolling Buffers (Normativ)

Minimale Buffergroessen je Pipeline:

| Buffer | Groesse | Entspricht |
|--------|---------|-----------|
| B1 (1m) | 240 Bars | ca. 4h |
| B3 (3m) | 200 Bars | ca. 10h (nur bei Decision-Layer 3m aktiv) |
| B5 (5m) | 120 Bars | ca. 10h (IMMER aktiv fuer fixe KPIs; bei Decision-Layer 5m auch Decision-Buffer) |
| Daily | 30 Tage | Fuer ADR14, Gap-Analyse |

**Verbraucher der Buffers:** QuantEngine, LLM Prompt Builder, Strategy Rules, Risk Monitor.

---

## 5. VWAP-Berechnung (Normativ)

VWAP ist **Source-of-Truth ausschliesslich in odin-data**. Die KPI-Engine konsumiert ihn aus dem MarketSnapshot.

Da kein Tick-VWAP verfuegbar ist, wird bar-basiert approximiert:

```
typical_price = (High + Low + Close) / 3
VWAP_day = sum(typical_price * Volume) / sum(Volume)
```

VWAP ist Bestandteil jedes MarketSnapshot und wird im EventLog versioniert. Die Berechnung wird ueber alle 1m-Bars des Tages kumuliert.

---

## 6. Data Quality Gates (Normativ)

Alle Marktdaten durchlaufen vor Einspeisung in den Rolling Buffer Quality Gates in **fester Reihenfolge**. Die Reihenfolge MUSS eingehalten werden.

### Gate 1: Bar Completeness

- OHLC vorhanden, Volume >= 0
- Unvollstaendige Bars verwerfen, `DQ_BAR_INCOMPLETE` Event

### Gate 2: Time Sync / Monotonicity

- barTimestamp MUSS strikt monoton sein
- System-Clock vs. Exchange-Timestamp Drift > 2s: Warnung an Monitoring

### Gate 3: Stale Bar Detection

- Keine neue 1m-Bar innerhalb von `stale_threshold` (Default: 120s bei aktivem Markt): `DQ_STALE` Warnung
- 30s keine Ticks: DQ-Gate-Warnung
- 60s keine Ticks: Kill-Switch via DQ-Gate (Stale Quote Detection)
- Wenn Position offen bei anhaltendem Stale: Kill-Switch/Forced-Exit (konfigurierbar)

### Gate 4: Crash Detection (markieren, NICHT verwerfen)

- Preis > `crash_pct` (Default: 5%) in < 1 Minute **UND** Volume > `crash_vol_ratio` (Default: 3.0x) Durchschnitt
- Emittiere `CRASH_DOWN` oder `SPIKE_UP` Event
- Bar bleibt im Buffer -- reale Extrembewegungen DUERFEN NICHT gefiltert werden

### Gate 5: Outlier Filter (nur wenn kein Crash-Signal)

- Preis > `outlier_move_pct` (Default: 5%) in < 1 Bar **UND** kein Crash-Signal **UND** Volume normal (< `outlier_max_volume_ratio`, Default: 1.2x)
- Vermutlich Datenfehler; Bar verwerfen, `DQ_OUTLIER_REJECTED` Event

### Gate 6: Gap Detection

- Open vs. vorheriger Close > Schwelle: `DQ_GAP` (realistisch, nicht verwerfen)
- Strategieregeln verschaerfen (z.B. keine Market-Entries, groessere Stops, kein Add)

### Gate 7: Session Boundary Guard

- Bars ausserhalb der Session werden je nach Regel (Pre/Post) akzeptiert oder ignoriert

> **Signal-Interferenz-Schutz:** Der Outlier-Filter prueft explizit, ob die Crash-Detection den Bar bereits als reale Extrembewegung klassifiziert hat. Nur Bars die sowohl eine extreme Preisabweichung zeigen ALS AUCH kein erhoehtes Volumen haben, werden als Outlier verworfen.

> **Degradation vor Kill-Switch:** Bei einzelnen Quality-Gate-Verletzungen wird das System zunaechst degradiert (groessere Intervalle, konservativere Regeln), bevor der Kill-Switch gezogen wird. Nur bei persistierendem Datenausfall (> 60s keine validen Ticks) wird sofort geschlossen.

### DQ-Gate Parameter (konfigurierbar)

| Parameter | Default | Beschreibung |
|-----------|---------|-------------|
| `dq_stale_seconds_warn` | 90s | Warnschwelle, keine Bar |
| `dq_stale_seconds_halt` | 180s | Halt-Schwelle |
| `crash_move_pct` | 5% | Move-Prozent fuer Crash/Spike |
| `crash_volume_ratio` | 3.0 | VolumeRatio fuer Crash/Spike |
| `outlier_move_pct` | 5% | Move-Prozent fuer Outlier |
| `outlier_max_volume_ratio` | 1.2 | Max. VolumeRatio fuer Outlier |

---

## 7. Feature-Katalog (OHLCV-Basis, stufenabhaengig erweiterbar)

### 7.1 Decision-Layer-Features (Decision-Bar-Timeframe: 3m oder 5m)

| Feature | Berechnung | Verwendung |
|---------|-----------|-----------|
| VWAP intraday | Aus 1m Volume, als Feature je Decision-Bar Snapshot | Mean-/Trend-Anchor |
| EMA(9) | Exponential Moving Average, Decision-Bars | Kurzfristiger Trend |
| EMA(21) | Exponential Moving Average, Decision-Bars | Trend-Kreuzung, Pullback-Qualitaet |
| Volumenratio | `vol_ratio = vol / SMA(vol, 20)` | Bestaetigung/Climax-Detektion |
| VWAP-Extension | `ext_vwap_atr = (close - vwap) / atr` | "Zu weit von VWAP" Pruefung |
| Struktur | Swing-High/Low, Higher-Low, Lower-High | Deterministisch, ohne Lookahead |

### 7.2 Fixe KPI-Features (IMMER 5m, unabhaengig vom Decision-Layer)

| Feature | Berechnung | Verwendung |
|---------|-----------|-----------|
| RSI(14) | Relative Strength Index, 5m | Overbought/Oversold-Veto, Momentum |
| ATR(14) | Average True Range, 5m | Stop-Distance, Sizing, Vol-Regime |
| ADX(14) | Average Directional Index, 5m | Trendqualitaet (Entry-Filter) |
| Bollinger Bands(20,2) | 5m | Ueberhitzung, Runner-Trailing, Extension |

### 7.3 Monitoring-Features (1m)

| Feature | Berechnung | Verwendung |
|---------|-----------|-----------|
| Jump/Crash Detection | `abs(return_1m)` + Volumenratio | Extrembewegungen erkennen |
| Wick Ratios (oben/unten) | `(high - max(open, close)) / (high-low)` | Rejection-Signale |
| Micro-Structure Proxy | Folge grosser Kerzen in eine Richtung, "climax bar" | Exhaustion-Erkennung |
| Bar-Completeness, Staleness, Outlier | DQ-Gate-Checks | Datenqualitaet |

### 7.4 Mehrtaegige Kontextfeatures (aus Daily Aggregation)

| Feature | Berechnung | Verwendung |
|---------|-----------|-----------|
| ADR(14) | Average Daily Range (14 Tage) aus Daily Aggregation der 1m Bars | "Ist das schon zu weit gelaufen?" |
| Relative Move | `intraday_range / ADR14` | Extension-Checks |
| Gap vs. Vortagsschluss | Wenn Vortag verfuegbar | Gap-Analyse |

### 7.5 Derived Values (Normativ)

| Wert | Formel | Verwendung |
|------|--------|-----------|
| ATR-Decay-Ratio | `current_ATR / morning_ATR` | Volatilitaetsabnahme im Tagesverlauf |
| Volume-Ratio | `lastBarVolume / SMA(barVolume, 20)` | Volumen-Bestaetigung |
| DistToVWAP_ATR | `(close - vwap) / ATR` | VWAP-Extension-Pruefung |

**ATR-Decay-Trigger:** Wenn `ATR-Decay-Ratio < 0.40` (d.h. aktuelle ATR ist auf weniger als 40% der Morning-ATR gefallen), MUSS die Strategie von Trend-Following auf konservativeres Verhalten umschalten: groessere Decision-Intervalle, kleinere Positionen, Mean-Reversion bevorzugen. Dieser Trigger wird dem LLM als Kontext mitgegeben.

---

## 8. Pattern-Feature-Erkennung (Normativ)

Aus den Rolling Buffers werden deterministische Pattern-Features berechnet, die als Input fuer die Trade-Pattern-State-Machines und das LLM dienen:

| Feature | Berechnung | Verwendung |
|---------|-----------|-----------|
| Flush-Detection | Bar-Range > 2x ATR(14) AND Close < Open AND Close nahe Bar-Low (< 20% der Range) AND Volume_Ratio > X | Trigger fuer Setup B (FLUSH_DETECTED) |
| Reclaim-Detection | Nach Flush: Preis > VWAP AND Preis > MA-Cluster innerhalb von 3 Bars | Uebergang FLUSH_DETECTED -> RECLAIM |
| Coiling-Detection | Fallende N-Bar-Hochs + steigende N-Bar-Tiefs + ATR(5) < 60% ATR(14) | Trigger fuer Setup C (COIL_FORMING) |
| Higher-Low-Detection | Lokales Minimum > vorheriges lokales Minimum (Swing-Low-Analyse, nur bestaetigte Swings, kein Lookahead) | Trendbestaetigung, Trailing-Stop-Anker |
| Breakout-Strength | Bar-Range bei Breakout vs. durchschnittliche Range der Coil-Phase | Validierung ob echter Breakout vs. Fakeout |
| Pullback-Depth | Ruecksetzer-Tiefe relativ zum vorherigen Impuls (in ATR-Einheiten) | Beurteilung ob Pullback "gesund" (< 0.5x ATR) oder strukturbrechend |

> **Swing-Analyse MUSS ohne Lookahead implementiert werden** (nur bestaetigte Swings).

---

## 9. 1m Event-Detektor (Eskalation)

Der 1m-Layer ist ein reiner Monitoring-/Event-Detektor fuer extreme Moves. Er erzeugt Events, die den Decision-Cycle auf dem Decision-Layer beeinflussen:

| Event | Erkennung | Wirkung |
|-------|-----------|---------|
| `CRASH_DOWN` | Preis > crash_pct nach unten + Volume Spike | Triggert LLM-Refresh, schaltet Risk Mode, priorisiert Exit |
| `SPIKE_UP` | Preis > crash_pct nach oben + Volume Spike | Triggert LLM-Refresh, Exhaustion-Check |
| `EXHAUSTION_CLIMAX` | Drei-Saeulen-Exhaustion auf 1m (Extension + Climax + Rejection) | Scale-Out aggressiver, Trail enger, Adds blockieren |
| `STRUCTURE_BREAK_1M` | Lower-Low / Close unter Vorbar-Low | Priorisiert Exit im naechsten Decision-Cycle |
| `DQ_STALE` | Keine neue 1m-Bar nach Schwelle | Trading Halt / Degradation |
| `DQ_OUTLIER` | Outlier-Bar erkannt | Bar verwerfen |
| `DQ_GAP` | Open vs. Close Gap | Strategieregeln verschaerfen |
| `VWAP_CROSS_1M` | Preis kreuzt VWAP auf 1m | Hinweis, kein direkter Trigger |
| `EMA_CROSS_1M` | EMA-Kreuzung auf 1m | Hinweis, kein direkter Trigger |

**Regel:** Der 1m-Monitor DARF NICHT final handeln, sondern nur:

- Stop-/Risk-Protect ausloesen
- Oder eine sofortige Decision Cycle anfordern (ausserhalb des Decision-Bar-Takts bei CRITICAL Events)

---

## 10. IB TWS API Subscriptions (Live-Modus)

| Feed | Subscription | Verwendung | Update-Frequenz |
|------|-------------|-----------|-----------------|
| Realtime Bars | `reqRealTimeBars()` | 5-Sekunden-Bars | Alle 5 Sekunden |
| Historical Data | `reqHistoricalData()` | 1-Min + 5-Min Bars fuer LLM-Kontext | Init + alle 5 Min Refresh |
| Market Depth (L2) | `reqMktDepth()` | Spread-Monitoring, Liquiditaets-Check (nur Live, optional) | Echtzeit |
| Tick Data | `reqMktData()` | Last Price, Bid/Ask, Volume | Echtzeit |
| Order Status | `reqOpenOrders()` | Fill-Bestaetigung, Order-Tracking | Event-basiert |
| Account Data | `reqAccountUpdates()` | Cash-Balance, Reconciliation-Backup | Alle 3 Min |
| P&L Stream | `reqPnL()` / `reqPnLSingle()` | Echtzeit unrealisierte P&L pro Position | Echtzeit (IB-Push) |

---

## 11. Datenkompression fuer LLM Context Window

| Zeitfenster | Granularitaet | Datenpunkte |
|-------------|--------------|-------------|
| Letzte 15 Minuten | 1-Minuten-Bars (OHLCV) | ca. 15 |
| Letzte 2 Stunden | 5-Minuten-Bars (OHLCV) | ca. 24 |
| Rest des Tages | 15-Minuten-Bars (OHLCV) | ca. 16 |
| Letzte 5 Handelstage | 30-Minuten-Bars | ca. 65 |
| Letzte 20 Handelstage | Daily Bars | 20 |

Zusaetzlich vorberechnete Aggregationen: VWAP, kumulatives Volumen-Profil, Intraday-High/Low, aktuelle RSI-Werte auf mehreren Timeframes, ATR-Decay-Ratio, Volume-Ratio.
