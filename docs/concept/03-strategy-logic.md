# 03 -- Strategielogik, FSM, Decision Loop, Setups, Multi-Cycle-Day

> **Quellen:** Fachkonzept v1.5 (Kap. 4--5: Tages-Lifecycle, Zustandsmaschine; Kap. 7: Deterministic Strategy Rules; Kap. 9: Decision Loop), Master-Konzept v1.1 (Strategielogik, Multi-Cycle-Day, Pattern-State-Machines), Build-Spec v3.0 (Kap. 7: FSM, Kap. 9: Strategy Rules), Strategie-Sparring (Setup-Definitionen A--D, Multi-Cycle-Guardrails, Re-Entry-Regeln), Stakeholder-Feedback (Entry-Cutoffs, Cooling-Off, Pre-Market-Trading)

---

## Begriffshierarchie (Normativ)

Die folgenden Begriffe werden im gesamten Dokument konsistent verwendet:

| Begriff | Definition | Beispiel |
|---------|-----------|---------|
| **Trade** | Erste Eroeffnung bis vollstaendige Schliessung einer Position in einem Instrument. Ein Trade kann aus mehreren Tranchen bestehen | Kauf AAPL 100 Stueck → spaeter Kauf 60 Stueck (Add) → Verkauf 80 Stueck → Verkauf 80 Stueck = **ein** Trade |
| **Cycle** | Ein vollstaendiger Trade im selben Instrument am selben Tag. Cycle 1 = erster Trade, Cycle 2 = Re-Entry nach vollstaendiger Schliessung | Erster AAPL-Trade (Cycle 1) → flat → Cooldown → Re-Entry (Cycle 2) |
| **Tranche** | Teilstueck innerhalb eines Trades. Entsteht durch gestuften Positionsaufbau (Starter/Add) oder gestufte Gewinnmitnahme (Scale-Out) | Starter-Tranche (40%) + Add-Tranche (60%) = eine Position, ein Trade |

> **Abgrenzung:** Ein **Add** (Aufstocken) innerhalb einer bestehenden Position erzeugt eine neue **Tranche**, aber keinen neuen **Cycle**. Ein neuer Cycle entsteht nur, wenn die Position vollstaendig geschlossen wurde und ein Re-Entry erfolgt.

---

## 1. Tages-Lifecycle

> **Zeitzone:** Alle Uhrzeiten in **US Eastern Time (ET)**. Die Implementierung MUSS Sommerzeitwechsel korrekt behandeln (Exchange-TZ `America/New_York`). Alle Phasen sind als konfigurierbare Parameter implementiert (Exchange-spezifisch).

| Phase | Zeitfenster (ET) | Verhalten |
|-------|-----------------|-----------|
| **Pre-Market** | 07:00 -- 09:30 | System-Initialisierung, Historische Daten laden (20d Daily + 5d Intraday), LLM-Tagesplan (Overnight-Gap-Analyse, erwartetes Regime, Opportunity Zones). Passive Beobachtung. Trades nur bei aktiviertem Pre-Market-Trading (halbe Position, verschaerfte Regeln) |
| **Opening Auction** | 09:30 -- 09:45 | Reine Diagnose. **Kein Entry.** ATR- und Volumen-Baseline kalibrieren. Opening-Spike-Erkennung aktiv |
| **Aktive Handelsphase** | 09:45 -- 15:15 | Decision Loop laeuft. Entries und Exits erlaubt. LLM wird periodisch konsultiert (3--15 Min je nach Volatilitaet) |
| **Power Hour** | 15:15 -- 15:45 | Verschaerfte Regeln. **Entries bis 15:30 erlaubt** (Erst-Entry), danach keine neuen Positionen. Enge Trailing-Stops. Time-based Tightening aktiv |
| **Forced Close** | 15:45 -- 15:55 | Alle offenen Positionen werden zwangsgeschlossen. Market-Orders erlaubt |
| **End-of-Day** | 15:55 -- 16:15 | Reconciliation, Performance-Logging, Ground-Truth-Labeling, Calibration-Updates |

---

## 2. Pipeline-Zustandsmaschine (FSM)

Jede Pipeline (= ein Instrument) durchlaeuft pro Handelstag die folgende Zustandsmaschine. Die FSM ist das zentrale Steuerungsmodell -- alle Entscheidungen sind an den aktuellen State gebunden.

### 2.1 State-Definitionen

| State | Beschreibung | LLM aktiv? | Trading erlaubt? |
|-------|-------------|------------|-----------------|
| **INITIALIZING** | Systemstart, Config-Snapshot erstellen, Daten laden, Buffer initialisieren | Ja (Planung) | Nein |
| **WARMUP** | Indikator-Warmup laeuft (period + 1 Bars je Frame). Decision Loop laeuft mit Guards, aber Entry blockiert | Ja (Analyse) | Nein (Entry blockiert) |
| **OBSERVING** | Markt beobachten, Regime diagnostizieren. Erstmaliges Beobachten ohne vorherigen Trade | Ja (Analyse) | Nein |
| **SEEKING_ENTRY** | Aktiv nach Entry-Punkt suchen. Setup-FSMs aktiv | Ja (Kontext) | Ja (Entry) |
| **PENDING_FILL** | Order gesendet, warte auf Fill. Repricing-Zyklen laufen (max. 3) | Nein | Nein (wartend) |
| **MANAGING_TRADE** | Position offen, Stop aktiv, Exit-Timing. Trailing-Stops, Scaling-Out, TACTICAL_EXIT | Ja (Kontext) | Ja (Exit/Adjust) |
| **FLAT_INTRADAY** | Position komplett geschlossen, Tag laeuft weiter. Traegt Zyklen-Kontext (realisierter P&L, verbrauchtes Risk-Budget, Cycle-Counter) | Ja (Analyse) | Ja (Re-Entry moeglich) |
| **COOLDOWN** | Nach Exit, Overtrade-Schutz. Mindestens 15 Minuten zwischen Zyklen | Nein | Nein |
| **DAY_STOPPED** | Tagesverlust-Limit erreicht, Max-Cycles erreicht, oder schwerer DQ-Fehler | Nein | Nein |
| **FORCED_CLOSE** | Handelszeit vorbei, Positionen schliessen | Nein | Nur Exits |
| **EOD** | Reconciliation, Performance-Logging, Ground-Truth-Labeling | Nein | Nein |

### 2.2 State-Uebergaenge

```
INITIALIZING → WARMUP → OBSERVING ↔ SEEKING_ENTRY → PENDING_FILL → MANAGING_TRADE
                                                                          │
                                                                   FLAT_INTRADAY
                                                                          │
                                                                       COOLDOWN
                                                                          │
                                                             → SEEKING_ENTRY (Re-Entry)
                                                             → FORCED_CLOSE
                                                             → DAY_STOPPED
```

**Detaillierte Transitionen:**

| Von | Nach | Bedingung |
|-----|------|-----------|
| INITIALIZING | WARMUP | Daten geladen, Config-Snapshot erstellt |
| WARMUP | OBSERVING | `warmupComplete = true` (alle Indikatoren gueltig) |
| OBSERVING | SEEKING_ENTRY | Regime erkannt (Confidence >= 0.6), Taktik gewaehlt |
| SEEKING_ENTRY | PENDING_FILL | Entry validiert (Quant + LLM + Risk Gate), Order gesendet |
| SEEKING_ENTRY | OBSERVING | Kein gutes Setup gefunden |
| PENDING_FILL | MANAGING_TRADE | Order gefuellt (Full Fill oder Partial Fill mit Restposition) |
| PENDING_FILL | SEEKING_ENTRY | Kein Fill nach max. Repricing-Zyklen, Order storniert |
| MANAGING_TRADE | FLAT_INTRADAY | Position komplett verkauft, Handelstag laeuft weiter |
| FLAT_INTRADAY | COOLDOWN | Cooling-Off-Timer startet (15 Min) |
| COOLDOWN | SEEKING_ENTRY | Cooling-Off abgelaufen, LLM: `ALLOW_RE_ENTRY`, cycleNumber < maxCycles |
| FLAT_INTRADAY | FORCED_CLOSE | Handelszeit endet |
| FLAT_INTRADAY | DAY_STOPPED | Hard-Stop oder Max-Cycles erreicht |
| MANAGING_TRADE | FORCED_CLOSE | Handelszeit endet (15:45 ET) |
| *Jeder State* | DAY_STOPPED | Hard-Stop erreicht (-10% Tageskapital) |
| *Jeder State* | FORCED_CLOSE | MarketClock > Forced-Close-Start |
| FORCED_CLOSE | EOD | Alle Positionen geschlossen |
| DAY_STOPPED | EOD | Tagesende |

**FLAT_INTRADAY vs. OBSERVING (Stakeholder-Entscheidung):** FLAT_INTRADAY traegt den Kontext des abgeschlossenen Zyklus (realisierter P&L, verbrauchtes Risk-Budget, Cycle-Counter). OBSERVING ist der initiale Zustand ohne diesen Kontext. Die Trennung ist notwendig, damit die Rules Engine bei Re-Entry-Pruefungen auf die Tageshistorie zugreifen kann.

**Multi-Cycle:** `cycleNumber` wird beim Uebergang COOLDOWN → SEEKING_ENTRY inkrementiert.

---

## 3. Decision Loop (3m Decision-Bar)

Der Decision Loop ist die deterministische Hauptschleife des Systems. Er wird bei jedem Close einer 3m-Bar ausgefuehrt und orchestriert alle Entscheidungsschritte.

### 3.1 Die 10 Schritte

Bei jedem 3m-Bar-Close:

| Schritt | Aktion | Verantwortlich |
|---------|--------|---------------|
| 1 | **Hard-Stop-Pruefung** -- Tagesverlust >= 10%? → DAY_STOPPED. Exposure-Limit, Max-Cycles pruefen | Risk |
| 2 | **Data Quality Gate** -- Bar Completeness, Staleness, Outlier, Crash-Check | Data |
| 3 | **MarketSnapshot erzeugen** (immutable, MarketClock-Timestamp). Snapshots fuer 1m/3m/10m | Data |
| 4 | **KPI-Engine** -- Indikatoren berechnen (EMA, RSI, ATR, ADX, VWAP, Bollinger, Volume-Ratio) → IndicatorResult. QuantVote erzeugen (Regime + Intent + Gate-Ergebnisse + Vetos) | Brain/Quant |
| 5 | **LlmVote** aus LlmAnalysisStore lesen (async bereitgestellt, TTL-Check). Falls stale/fehlend: kein Entry moeglich | Brain/LLM |
| 6 | **Regime-Fusion** -- Quant + LLM + 10m Confirmation + Hysterese (2 konsekutive 3m-Bars fuer Wechsel). Bei Widerspruch: konservativeres Regime | Brain/Arbiter |
| 7 | **Strategy Rules** -- Setup-FSMs evaluieren, Entry/Exit-Bedingungen pruefen, Pattern-Gates pruefen | Brain/Rules |
| 8 | **Decision Arbiter** -- Votes kombinieren (Dual-Key), Konflikte aufloesen → TradeIntent oder NONE | Brain/Arbiter |
| 9 | **Risk Gate** -- Position Sizing, Budget-Check, R/R-Ratio, Stop-Distance-Bounds | Risk |
| 10 | **OMS** -- Order erzeugen, Stops/Targets platzieren, oder Intent verwerfen + EventLog | Execution |

### 3.2 Ein-Intent-pro-Decision-Bar

Maximal ein TradeIntent pro Decision-Cycle. Bei Tie-Break: **Exit > Entry**. Trailing-Stops und Scaling-Out sind OMS-Maintenance, keine TradeIntents.

### 3.3 Reject-Semantik

| Ablehnungsgrund | Reaktion | Retry? |
|-----------------|----------|--------|
| Quant Hard-Veto (Spread, RSI, EMA) | Intent verwerfen. Kein Retry fuer diesen Trigger-Zyklus | Nein |
| Quant Gate-Fail (mindestens ein Gate nicht bestanden) | Intent verwerfen. Naechster regulaerer Trigger-Zyklus darf neu evaluieren | Nein (naechster Zyklus) |
| Risk Gate: Position zu gross | Intent verwerfen. Kein automatisches Downsizing | Nein |
| Risk Gate: Tages-Budget erschoepft | Intent verwerfen. Keine weiteren Entries heute | Nein (final) |
| Risk Gate: R/R-Ratio unzureichend | Intent verwerfen. Naechster Zyklus darf neu evaluieren | Nein (naechster Zyklus) |
| LLM stale/fehlend (Entry) | Entry blockiert. Bestehende Positionen weiter verwaltet | Nein |
| Dual-Key-Fail | Intent verwerfen. ReasonCode: `HYBRID_DUAL_KEY_FAIL` | Nein |

**Kein automatisches Retry.** Jeder abgelehnte Intent wird final verworfen und im Audit-Log mit ReasonCode protokolliert. Retry-Schleifen wuerden zu Overtrading und Signal-Fixierung fuehren.

### 3.4 1m Monitor Loop (Eskalation)

Zusaetzlich zum 3m Decision Loop laeuft ein 1m Monitoring Loop. Er DARF NICHT final handeln, sondern nur:

- Stop-/Risk-Protect ausloesen (OMS-Maintenance)
- Eine sofortige Decision-Cycle-Anforderung bei CRITICAL Events (ausserhalb des 3m-Takts)
- LLM-Refresh triggern

| Event | Erkennung | Wirkung |
|-------|-----------|---------|
| `CRASH_DOWN` | Preis > crash_pct nach unten + Volume Spike | Triggert LLM-Refresh, schaltet Risk Mode, priorisiert Exit |
| `SPIKE_UP` | Preis > crash_pct nach oben + Volume Spike | Triggert LLM-Refresh, Exhaustion-Check |
| `EXHAUSTION_CLIMAX` | Drei-Saeulen-Exhaustion auf 1m (Extension + Climax + Rejection) | Scale-Out aggressiver, Trail enger, Adds blockieren |
| `STRUCTURE_BREAK_1M` | Lower-Low / Close unter Vorbar-Low | Priorisiert Exit im naechsten 3m Cycle |
| `DQ_STALE` | Keine neue 1m-Bar nach Schwelle | Trading Halt / Degradation |
| `DQ_OUTLIER` | Outlier-Bar erkannt (extrem + kein Volume) | Bar verwerfen |
| `DQ_GAP` | Open vs. Close Gap ueber Schwelle | Strategieregeln verschaerfen |
| `VWAP_CROSS_1M` | Preis kreuzt VWAP auf 1m | Hinweis, kein direkter Trigger |
| `EMA_CROSS_1M` | EMA-Kreuzung auf 1m | Hinweis, kein direkter Trigger |

### 3.5 10m Confirmation Loop

Bei jedem Close einer 10m-Bar:

- Bestaetigt Trend/Regime (Hysterese: 2 konsekutive 10m-Bars)
- Aktualisiert RegimePersistenz
- Setzt DayType-Flags (TREND_DAY_UP, TREND_DAY_DOWN, RANGE_DAY, HIGH_VOL_DAY)
- Wirkt als Gate im Arbiter: 10m Confirmation DARF NICHT gegensaetzlich zum Decision-Bar-Regime sein

---

## 4. Entry-Regeln

### 4.1 Mandatory Gates (hart, vor jedem Entry)

Entry ist blockiert, wenn:

- Opening Buffer aktiv (vor 09:45 ET)
- Warmup unvollstaendig (`warmupComplete = false`)
- Data Quality nicht ok (DQ-Gate-Verletzung)
- Tages-DD oder Max-Cycles erreicht
- Liquiditaetsproxy zu schwach (Volume < Schwelle, 0-Volume-Bars)
- Regime in `TREND_DOWN` / `UNCERTAIN` (je nach Policy)
- Entry Cutoff ueberschritten (nach 15:30 ET kein Erst-Entry; nach 15:15 ET kein Re-Entry)
- Kein frischer LLM-Output vorhanden (TTL < 120s fuer Entry)

### 4.2 Regime-Confidence-Schwellen

Ein frischer LLM-Output mit `regimeConfidence > 0` MUSS vorliegen. Ohne frischen LLM-Output ist `regimeConfidence = 0.0` und Entries sind blockiert. Das LLM **triggert** keinen Entry -- es liefert ein notwendiges Input-Feature.

| Regime-Confidence | Quant-Gate-Anforderung |
|-------------------|------------------------|
| < 0.5 | Kein Entry (Regime = UNCERTAIN) |
| 0.5 -- 0.7 | Alle Gates bestanden + verschaerfte Schwellen (siehe Gate-Kaskade) |
| > 0.7 | Alle Gates bestanden (Standard-Schwellen) |

### 4.3 Quant Gate-Kaskade (Normativ)

Die Quant-Validierung verwendet **keine gewichteten Scores**. Stattdessen wird eine harte Gate-Kaskade eingesetzt: Jeder Indikator hat eine Ja/Nein-Schwelle (Gate). **Alle** Gates muessen bestanden sein fuer Entry-Freigabe.

| Gate | Indikator | Standard-Schwelle | Verschaerfte Schwelle (Confidence 0.5--0.7) | Veto-Typ |
|------|-----------|-------------------|---------------------------------------------|----------|
| **Spread-Gate** | Bid-Ask Spread | < 0.5% | < 0.3% | Hard-Veto |
| **Volume-Gate** | Volume-Ratio (aktuell vs. SMA) | > 0.8x | > 1.0x | Hard-Veto |
| **RSI-Gate** | RSI(14) | 25 < RSI < 70 | 30 < RSI < 65 | Hard-Veto |
| **EMA-Trend-Gate** | EMA(9) vs. EMA(21) | EMA(9) > EMA(21) (Long) | Zusaetzlich EMA(21) > EMA(50) | Soft-Gate |
| **VWAP-Gate** | Preis relativ zu VWAP | Preis <= VWAP + 0.5x ATR(14) | Preis <= VWAP + 0.3x ATR(14) | Soft-Gate |
| **ATR-Gate** | ATR vs. Daily-ATR | ATR(14) > 0.5x Daily-ATR | ATR(14) > 0.6x Daily-ATR | Soft-Gate |
| **ADX-Gate** | ADX(14) | > 15 (Trend vorhanden) | > 20 (staerkerer Trend) | Soft-Gate |

**Hard-Veto vs. Soft-Gate:**

- **Hard-Veto:** Blockiert Entry sofort und endgueltig fuer diesen Decision-Cycle. Kein Override moeglich
- **Soft-Gate:** Muss bestanden sein, aber die Schwelle wird durch Regime-Confidence moduliert

**Kein gewichteter Score, keine Normalisierung.** Die Gate-Schwellen werden initial konservativ gesetzt und durch Backtests kalibriert. Jedes Gate liefert PASS oder FAIL -- es gibt keine Teilergebnisse.

> **Abgrenzung zum bisherigen Modell:** Der fruehere gewichtete `quant_score` (0.0--1.0) wurde durch die Gate-Kaskade ersetzt. Vorteil: Transparenz (jedes Gate hat einen klaren Grund fuer PASS/FAIL), keine versteckten Kompensationseffekte (ein schlechter Indikator kann nicht durch einen guten "ausgeglichen" werden).

### 4.4 Entry-Bedingungen pro Regime

| Regime | Entry-Bedingung | Quant-Gate-Anforderung |
|--------|----------------|------------------------|
| **TREND_UP** | EMA(9) > EMA(21) AND RSI < 70 AND Preis <= VWAP + 0.5x ATR(14) | Alle Gates bestanden (siehe Gate-Kaskade) |
| **TREND_DOWN** | **Default: Kein Entry.** Aggressive Mode (Config OFF): RSI < 25, Volume > 2x SMA, halbe Position, max. 2 Trades/Tag | Volumen-Spike + enge Stops (1.0x ATR statt 2.2x) |
| **RANGE_BOUND** | RSI < 40 AND Preis nahe VWAP-Support | Alle Gates bestanden (siehe Gate-Kaskade) |
| **HIGH_VOLATILITY** | Volume > 1.5x SMA AND Spread-Proxy < 0.3% | Sehr selektiv, halbe Position |
| **UNCERTAIN** | **Kein Entry** | -- |
| **OPENING_VOLATILITY** | **Kein Entry** (Diagnose-Phase) | -- |
| **EXHAUSTION_RISK** | **Kein Entry** (spaet im Trend) | -- |
| **RECOVERY** | Konservativer Entry (nach Flush/Korrektur) | Verschaerfte Gate-Schwellen |
| **BREAKOUT_ATTEMPT** | Entry bei bestaetiger Range-Expansion + Volume | 10m Confirmation nicht dagegen |

### 4.5 Dual-Key-Anforderung fuer Entry

Entry MUSS erfuellt sein (Symmetric Hybrid Protocol):

- Keine Quant Hard-Vetos
- LLM nicht `SAFETY_VETO`
- QuantVote in {`ALLOW_ENTRY`, `STRONG_ENTRY`}
- LlmVote in {`ALLOW_ENTRY`, `STRONG_ENTRY`}
- Confidences beider Seiten >= `min_conf_entry`
- 10m Confirmation nicht gegensaetzlich

---

## 5. Exit-Regeln (Prioritaetsordnung)

Exit-Trigger werden in fester Rangfolge geprueft. Hoehere Prioritaet uebersteuert niedrigere:

| Prio | Trigger | LLM-Abhaengigkeit | Detail |
|------|---------|--------------------|--------|
| 1 | **Stop-Loss** (Preis <= Entry - 2.2 x entryAtr) | Keine (Broker-seitig, GTC) | ATR bei Entry eingefroren, Faktor 2.2 |
| 2 | **Forced Close** (MarketClock > 15:45 ET) | Keine | Zeitbasiert, deterministisch |
| 3 | **Kill-Switch** | Keine | Notfall, sofort Market-Order |
| 4 | **Trailing-Stop** (Preis faellt um Trail-Factor x ATR vom High) | Keine | Highwater-Mark, R-basierte Stufen |
| 5 | **Regime-Wechsel** (2x bestaetigt, Position passt nicht) | Wuenschenswert | Beschleunigter Exit bei Regimewechsel |
| 6 | **TACTICAL_EXIT** (LLM exit_bias + KPI-Bestaetigung) | Nur als Signal | LLM allein erzwingt KEINEN Exit |
| 7 | **Zeitbasiert** (hold_duration_bars ueberschritten ODER < 30 Min bis Forced Close) | Kontext | Zeitlimit erreicht |
| 8 | **Gewinnmitnahmen** (R-basierte Tranchen) | Keine | Deterministische Target-Orders |

### 5.1 TACTICAL_EXIT (Stakeholder-Entscheidung)

TACTICAL_EXIT ist eine Exit-Prioritaet zwischen REGIME_CHANGE und zeitbasiertem Exit. Es ist der primaere Mechanismus fuer LLM-Einfluss auf Exit-Timing.

**KPI-Signale (alle aus IndicatorResult):**

| Signal | Bedingung |
|--------|-----------|
| `rsi_reversal` | RSI war > 70, kreuzt unter 60 |
| `ema_bear_cross` | EMA(9) < EMA(21) |
| `vwap_loss` | Close < VWAP bei hoher volumeRatio |
| `structure_break` | Lower-Low / Close unter Vorbar-Low |

**Logik nach exit_bias:**

| exit_bias | Verhalten |
|-----------|-----------|
| `NEUTRAL` / `HOLD` | TACTICAL_EXIT deaktiviert |
| `EXIT_SOON` | Exit wenn mind. 2 von 4 Signalen true ODER 1 Signal + exhaustion >= `EARLY_WARNING` |
| `EXIT_NOW` | Exit wenn mind. 1 Signal true UND (exhaustion == `CONFIRMED` ODER MFE >= 1R) |

> **Eiserne Regel:** Das LLM allein erzwingt KEINEN Exit. Ohne mindestens ein bestaetigendes KPI-Signal bleibt die Position offen, unabhaengig vom `exit_bias`.

### 5.2 Exit bei Dual-Key (Symmetric Hybrid Protocol)

- **Hard-Risk** (Stop, Kill, Forced Close): sofort, nicht verhandelbar
- Wenn **eine** Seite (Quant oder LLM) `EXIT_NOW` → setze `PENDING_EXIT`
- Exit wird ausgefuehrt, wenn innerhalb von 1--2 3m-Bars bestaetigt **ODER** 1m-Event (`EXHAUSTION_CLIMAX` / `STRUCTURE_BREAK_1M`) eintritt
- Wenn nicht bestaetigt: `EXIT_SOON`-Profile (Trail tighter, Scale-Out aggressiver)

---

## 6. Die vier Setups (Pattern-State-Machines)

Alle Setups existieren als FSM, die auf 3m-Entscheidungslogik basiert und 1m-Ereignisse als Trigger nutzt. Pattern-Aktivierung ist hybrid: LLM schlaegt Pattern-Kandidaten vor (`pattern_candidates` mit Enum + Confidence + Phase), aber State-Machine-Uebergaenge werden ausschliesslich durch KPI-Kriterien getriggert. Pattern-Confidence >= 0.5 UND deterministische KPI-Bestaetigung MUESSEN beide vorliegen.

### 6.1 Setup A -- Opening Consolidation → Trend Entry

**Ziel:** Richtiger Einstiegszeitpunkt nach Open, wenn erste High-Vol-Phase vorbei ist.

**States:**

| State | ReasonCode | Erkennungskriterien | Uebergang |
|-------|-----------|---------------------|-----------|
| `A_OBSERVE_OPEN` | `A_OBSERVE_OPEN` | Opening-Spike erkannt (1m Range/ATR hoch, VolumeRatio hoch) | → `A_WAIT_CONSOLIDATION` nach Opening Buffer |
| `A_WAIT_CONSOLIDATION` | `A_WAIT_CONSOLIDATION` | Zeitgate: `t >= open + opening_buffer_minutes`. ATR-Decay: `atr_current / atr_opening_baseline < threshold`. Range-Contraction: 2 von 3 letzten 3m Bars kleiner | → `A_READY` bei Konsolidierung bestaetigt |
| `A_READY` | `A_READY` | Konsolidierung abgeschlossen, VWAP-Akzeptanz (Preis nahe/ueber VWAP, keine massiven Rejects) | → `A_IN_TRADE` bei Entry-Signal |
| `A_IN_TRADE` | `A_IN_TRADE` | Position eroeffnet. Scale-Out bei 1R/2R oder Parabolic/Exhaustion Event | → `A_DONE` bei Exit |
| `A_DONE` | `A_DONE` | Setup abgeschlossen | → zurueck zu Pipeline-FSM |

**Entry-Typen innerhalb Setup A:**

1. **Pullback-Entry:** Regime TREND_UP oder BREAKOUT_ATTEMPT. Pullback in VWAP/EMA-Zone. 1m zeigt Selling-Pressure-Decay (kleinere rote Kerzen, lower volume). 3m Close zurueck ueber EMA(9).

2. **Breakout-Entry:** 3m Konsolidierungsrange wird gebrochen. 3m Range-Expansion > 1.5x ATR(3m). Volume-Ratio > Threshold. 10m Confirmation nicht dagegen.

**Add-Regeln:** Add nur wenn nach Entry hoehere Tiefs bestaetigt, kein Exhaustion-Risk, 1m Monitor kein `EXHAUSTION_CLIMAX`.

### 6.2 Setup B -- Flush → Reclaim → Run (V-Reversal)

**Ziel:** Scharfer Abverkauf (Stop-Run/Flush) gefolgt von schneller Rueckeroberung (Reclaim). Edge-Schablone, die stark ist wenn Kontext und Struktur passen, aber durchschnittlich bis schlecht performt wenn der Markt keinen Trend hergibt.

**States:**

| State | ReasonCode | Erkennungskriterien | Uebergang |
|-------|-----------|---------------------|-----------|
| `B_NEUTRAL` | -- | Kein Flush erkannt | → `B_FLUSH_DETECTED` bei Flush |
| `B_FLUSH_DETECTED` | `B_FLUSH_DETECTED` | Bar-Range > 2x ATR(14), Close < Open, Close nahe Bar-Low (< 20% der Range), Volume_Ratio > X, Sweep unter Key-Level (Tages-Low, VWAP) | → `B_RECLAIM` wenn Preis innerhalb 3 Bars > VWAP/MA-Zone. Timeout: Setup invalid |
| `B_RECLAIM` | `B_RECLAIM` | Close > VWAP UND Close > MA-Cluster, EMA(9) beginnt zu drehen, erste Higher-Low Struktur sichtbar. Hold min. 2 Bars | → `B_CONFIRM_RUN` bei Higher-Low + EMA(9) > EMA(21). **Starter-Entry** |
| `B_CONFIRM_RUN` | `B_CONFIRM_RUN` | Higher-Low bestaetigt, 10m Trendfilter dreht | → `B_IN_TRADE`. **Add-Position** bei Bestaetigung |
| `B_IN_TRADE` | `B_IN_TRADE` | Hoehere Tiefs, flache Pullbacks (< 0.5x ATR), Preis > Trend-MA | → `B_DONE` bei Lower-Low oder Close unter MA-Zone |
| `B_DONE` | `B_DONE` | Strukturbruch (Lower-Low) ODER Close unter MA-Cluster ODER Trailing-Stop | → zurueck zu Pipeline-FSM |

**Invalidierungslevel:** Das Low des Flush. Wird dieses unterschritten, ist das Setup sofort ungueltig.

**Failure Mode -- Bull Trap:** Preis reclaimed, wird sofort abverkauft → Exit klein via Initial-Stop (unter Flush-Low). Kein Re-Entry fuer dieses Pattern heute.

### 6.3 Setup C -- Coil/Compression → Breakout

**Ziel:** Aus enger Konsolidierung (symmetrisches Dreieck, Pennant, Keil) einen explosiven Breakout handeln.

**States:**

| State | ReasonCode | Erkennungskriterien | Uebergang |
|-------|-----------|---------------------|-----------|
| `C_NEUTRAL` | -- | Keine Kompression erkannt | → `C_COIL_FORMING` |
| `C_COIL_FORMING` | `C_COIL_FORMING` | Fallende Hochs + steigende Tiefs ueber min. 5 Bars, ATR faellt (ATR(5) < 60% ATR(14)), Range schrumpft, Volumen moderat/fallend | → `C_BREAKOUT_TRIGGERED` bei Kontraktion > 40% |
| `C_BREAKOUT_TRIGGERED` | `C_BREAKOUT_TRIGGERED` | Range < 60% der initialen Range, Preis nahe Apex. Close ausserhalb oberer Begrenzung + Bar-Range > 1.5x ATR + VolumeRatio > Threshold | → `C_RETEST_OPTIONAL` oder `C_IN_TRADE` |
| `C_RETEST_OPTIONAL` | `C_RETEST_OPTIONAL` | LLM Fakeout-Risk Bias bestimmt: Retest vs. Sofortentry | → `C_IN_TRADE` |
| `C_IN_TRADE` | `C_IN_TRADE` | Trend etabliert, Pullbacks halten Breakout-Level. Schnelle Profit-Protection (Coil fuehrt oft zu Blow-offs) | → `C_DONE` bei Reentry in Formation oder Lower-Low |
| `C_DONE` | `C_DONE` | Rueckkehr in Formation ODER Trailing-Stop | → zurueck zu Pipeline-FSM |

**Fakeout-Schutz:** Je naeher am Apex, desto hoeher das Fakeout-Risiko. Am Apex wird keine neue Position eroeffnet -- Entry nur bei Breakout mit klarer Bestaetigung (Close ausserhalb + Range). Risiko am Apex reduzieren, Markt "zeigen lassen" wohin er will.

**Distribution-Warnsignale:** Untere Trendlinie wird weich (Tiefs nicht verteidigt), Erholungen schwaecher, Close unter unterer Linie + unter MAs = Regimewechsel.

### 6.4 Setup D -- Re-Entry (neuer Cycle) nach Korrektur

**Ziel:** Nach vollstaendigem Exit (Cycle abgeschlossen) oder nach fruehm Downtrend spaeteren Trendwechsel handeln.

**States:**

| State | ReasonCode | Erkennungskriterien | Uebergang |
|-------|-----------|---------------------|-----------|
| `D_ELIGIBLE` | `D_ELIGIBLE` | Nur wenn Cycle > 0 oder Vormittag ohne Trade. Pipeline in FLAT_INTRADAY oder OBSERVING | → `D_BOTTOMING` |
| `D_BOTTOMING` | `D_BOTTOMING` | Korrektur hat Struktur (Lower-Lows enden, Higher-Low entsteht). Volumen-Climax (Selling Exhaustion) | → `D_RECLAIM` |
| `D_RECLAIM` | `D_RECLAIM` | VWAP reclaim + 10m Trend kippt oder stabilisiert. LLM sieht "Re-Accumulation" statt "Distribution" | → `D_CONFIRM` |
| `D_CONFIRM` | `D_CONFIRM` | Higher-Low bestaetigt, EMA-Rekreuzung, Volumen-Shift | → `D_IN_TRADE` |
| `D_IN_TRADE` | `D_IN_TRADE` | Konservativeres Profil: kleinere Position, engerer Stop | → `D_DONE` |
| `D_DONE` | `D_DONE` | Exit oder EOD | → zurueck zu Pipeline-FSM |

**Gates:** Cooldown abgelaufen, 10m nicht DOWN confirmed, Reclaim VWAP + Higher-Low, LLM erkennt Regimewechsel (Bias Shift). Sizing reduziert (60--80% von Cycle 1), aggressivere Profit Protection.

---

## 7. Position Building: Tranchen (Starter + Add)

Fuer Setup B (und analog fuer andere Setups) wird eine gestufte Positionsaufbau-Strategie ueber **Tranchen** verwendet. Jede Tranche hat eigene Entry-Parameter, aber alle Tranchen gehoeren zum selben Trade.

| Tranche | Anteil | Zeitpunkt | Stop |
|---------|--------|-----------|------|
| **Starter-Tranche** | ~40% der Zielgroesse | RECLAIM-State (erste Bestaetigung) | Unter Flush-Low (weit) |
| **Add-Tranche** | ~60% der Zielgroesse | RUN-State bestaetigt (Higher-Low, EMA-Kreuzung) | Unter letztes Higher-Low (enger) |

**Gesamtrisiko:** Alle Tranchen eines Trades DUERFEN den `max_risk` pro Trade (3% Tageskapital) NICHT ueberschreiten.

**Pattern-Aktivierung (Normativ):** LLM schlaegt Patterns vor (`patternCandidates` mit Enum + Confidence + Phase), aber State-Machine-Uebergaenge werden ausschliesslich durch KPI-Kriterien getriggert. Mehrere Patterns koennen parallel ueberwacht werden, aber nur ein Pattern darf gleichzeitig eine Position halten.

---

## 8. Multi-Cycle-Day (Stakeholder-Entscheidung)

### 8.1 Paradigmenwechsel

ODIN unterstuetzt **mehrere vollstaendige Entry/Exit-Zyklen pro Tag** auf demselben Instrument. Dies ist DER zentrale Use-Case fuer die LLM-Integration -- ein statischer Algorithmus kann nicht zuverlaessig entscheiden, ob ein Plateau temporaer oder terminal ist.

> **Kernthese (Stakeholder-Entscheidung):** ODIN ist KEIN reines Algo-Trading-System. Das LLM MUSS unterstuetzende Entscheidungsgewalt haben -- gebunden an deterministische Leitplanken.

### 8.2 Trade-Typen pro Cycle

| Trade-Typ | Cycle | Timing | Positionsgroesse | Besonderheit |
|-----------|-------|--------|-----------------|--------------|
| **Trend-Riding** | Cycle 1 | Fruehe RTH | Standard | Proaktiver Exit beim Plateau via TACTICAL_EXIT. Exit bei Exhaustion statt Warten auf Trailing-Stop |
| **Recovery-Trade** | Cycle 2+ | Nach Korrektur (typ. 30--90 Min) | Reduziert (Default 60%) | Konservativere Entry-Schwellen, engerer initialer Stop |
| **Scalp/Quick-Trade** | Beliebig | Klar kurzfristig | Klein, enge Stops | Explizit als spaeteres Feature (P4d) |

### 8.3 Re-Entry (neuer Cycle) vs. Aufstocken (neue Tranche)

| Merkmal | Re-Entry (neuer Cycle) | Aufstocken (neue Tranche) |
|---------|------------------------|---------------------------|
| Position zwischen Zyklen | 0 (komplett flat) | > 0 (Runner gehalten) |
| Pipeline-State | FLAT_INTRADAY | MANAGING_TRADE (unveraendert) |
| Trigger | `entry_timing_bias = ALLOW_RE_ENTRY` | LLM-Recovery-Signal + KPI bei bestehender Position |
| Semantik | Neuer Trade (neuer Cycle) | Vergroesserung einer bestehenden Position (neue Tranche, selber Trade) |

**Scale-Up OMS-Implikationen:**

- Aufgestockter Teil hat eigenen Entry-Preis und Entry-ATR (zum Zeitpunkt des Aufstockens)
- Position besteht aus Tranchen mit unterschiedlichen Entry-Parametern
- P&L-Berechnung pro Tranche
- Aufstocken erzeugt eine neue Tranche, aber **keinen** neuen Cycle (Position war nie flat)

### 8.4 Guardrails fuer Multi-Cycle

| Guardrail | Regel | Begruendung |
|-----------|-------|-------------|
| Max-Cycles-Per-Day | 3 (konfigurierbar), Hard-Cap als Guardrail | Verhindert Overtrading |
| Cooling-Off nach Exit | Min. 15 Minuten zwischen Zyklen | Verhindert Impuls-Re-Entry |
| Profit-Gate | Re-Entry nur wenn vorheriger Zyklus profitabel | Kein "Nachlegen" nach Verlust |
| Budget-Gate | Verbleibendes Risk-Budget MUSS minimale Position erlauben | Keine Micro-Entries |
| LLM-Pflicht | Re-Entry nur mit frischem LLM-Assessment (`ALLOW_RE_ENTRY`) | Kein algorithmischer Re-Entry ohne LLM-Kontext |
| Zeitfenster | Kein Re-Entry nach 15:15 ET (30 Min vor FORCED_CLOSE) | **Strenger als Erst-Entry** (bis 15:30 erlaubt) |
| Globales Limit | Max. 5 Cycles/Tag global ueber alle Pipelines | Dominiert ueber Per-Pipeline-Limit |

### 8.5 LLM-Rolle bei Multi-Cycle

Das LLM MUSS vier Arten von Entscheidungen unterstuetzen:

1. **Plateau-Erkennung** -- Exhaustion-Signale fuer proaktiven Exit (via TACTICAL_EXIT): Widerstandsniveaus, Erschoepfungsmuster, nachlassender Orderflow
2. **Drawdown-Timing** -- Bewertung ob Korrektur abgeschlossen ist: "Abverkauf noch aktiv" vs. "Boden gefunden"
3. **Recovery-Signal** -- Identifikation nachhaltiger Erholung: V-Recovery, konstruktive Konsolidierung, Kontext-Wechsel
4. **Re-Entry-Freigabe** -- `entry_timing_bias = ALLOW_RE_ENTRY` mit spezifischen Guards

---

## 9. Strategie-Policy Map (Regime → erlaubte Aktionen)

| Regime | Primaerziel | Entries? | Adds? | Scale-Out? | Re-Entry? |
|--------|------------|---------|-------|-----------|----------|
| OPENING_VOLATILITY | Diagnose | Nein | Nein | Nur Risk | Nein |
| TREND_UP | Trend reiten | Ja | Ja | Standard/Trail | Ja (nach Korrektur) |
| RANGE_BOUND | Kapital schuetzen | Selten | Nein | Klein/defensiv | Selten |
| HIGH_VOLATILITY | Ueberleben | Sehr selektiv | Nein | Aggressiv (Risk) | Erst nach Beruhigung |
| EXHAUSTION_RISK | Gewinne schuetzen | Nein (spaet) | Nein | Aggressiv | Ggf. spaeter |
| RECOVERY | Turn nutzen | Ja (konservativ) | Selektiv | Standard | Ja |
| BREAKOUT_ATTEMPT | Expansion nutzen | Ja | Ja (bei Bestaetigung) | Standard | Ja |
| TREND_DOWN | Long-only: vermeiden | Nein | Nein | Exit/flat | Nein |
| UNCERTAIN | Warten | Nein | Nein | Nur Risk | Nein |

---

## 10. Pre-Market-Trading

Pre-Market-Trading ist per Konfigurationsschalter steuerbar (`odin.core.pipeline.pre-market-trading-enabled`, **Default: false**). Bei aktiviertem Schalter darf der Agent waehrend der Pre-Market-Phase von OBSERVING nach SEEKING_ENTRY wechseln.

| Regel | Wert |
|-------|------|
| Max. Positionsgroesse | 25% Tageskapital (halbe Standardgroesse) |
| Mindest-Spread | < 0.5% (Bid-Ask relativ zum Midpoint) |
| Mindest-Volumen | > 10.000 Stueck in den letzten 30 Min Pre-Market |
| Repricing | Kein Repricing -- ein Limit-Order-Versuch |

---

## 11. Instrument-Selektion

Instrumente werden **extern vorgegeben** und beim Start der Pre-Market-Phase (07:00 ET) eingefroren. Kein Symbolwechsel waehrend des Handelstages. Die Auswahl erfolgt ausserhalb des Agenten (manuell oder durch vorgelagertes Screening). ODIN handelt nur vorgegebene Symbole -- es gibt kein internes Instrument-Discovery.

**Anforderungen an geeignete Instrumente:**

- Aktien (US, Europa)
- Ausreichende Liquiditaet (Spread < 0.5%, taegl. Volumen > 500.000)
- Keine Penny Stocks (Preis > 5 USD/EUR)
- Handelbar ueber IB TWS
