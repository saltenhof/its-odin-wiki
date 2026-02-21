# ODIN — Gesamtkonzept (Master Concept)

Version: 1.1
Stand: 2026-02-20
Konsolidiert aus: Fachkonzept v1.5, Strategie-Optimierung-Sparring, Architekturkapitel 0-10, Guardrails

> **Kapitelreferenzen:** Dieses Dokument verwendet **MC-1 bis MC-11** fuer seine eigenen Kapitel. Verweise auf die Architektur-Detailkapitel verwenden das Praefix **ARCH-** (z.B. ARCH-00 fuer Systemuebersicht, ARCH-07 fuer OMS). So wird eine Verwechslung zwischen Master-Concept-Kapitel 7 (Risk Management) und Architektur-Kapitel 07 (OMS) vermieden.

> **Normative Sprache:** Dieses Dokument verwendet RFC-2119-konforme Modalverben: **MUSS** = zwingend erforderlich, **DARF NICHT** / **NIE** = verboten, **SOLL** = dringend empfohlen (Abweichung nur mit dokumentierter Begruendung), **KANN** = optional. **IMMER** wird synonym zu MUSS verwendet. Regeln ohne Modalverb sind deskriptiv und beschreiben den Ist-Zustand oder Design-Intent.

> **Status-Kennzeichnung:** Regeln und Features sind mit ihrem Implementierungsstatus gekennzeichnet: **(Normativ)** = verbindlich und implementiert oder bei Implementierung zwingend einzuhalten, **(Geplant)** = konzipiert aber noch nicht implementiert, **(Stakeholder-Entscheidung)** = durch den Stakeholder entschieden und vorrangig gegenueber aelteren Dokumenten.

---

## Inhaltsverzeichnis

1. [Executive Summary](#1-executive-summary)
2. [Strategielogik](#2-strategielogik)
3. [Multi-Cycle-Day](#3-multi-cycle-day)
4. [Trailing Stop und Profit Protection](#4-trailing-stop-und-profit-protection)
5. [LLM-Integration](#5-llm-integration)
6. [OMS](#6-oms)
7. [Risk Management](#7-risk-management)
8. [KPI Engine](#8-kpi-engine)
9. [Datenmodell](#9-datenmodell)
10. [Frontend](#10-frontend)
11. [Offene Punkte / Roadmap](#11-offene-punkte-roadmap)

---

## 1. Executive Summary

### Was ist ODIN?

**ODIN** (Orderflow Detection & Inference Node) ist ein vollautomatischer Intraday-Trading-Agent fuer Aktien (US, Europa). Das System kombiniert drei Saeulen:

1. **LLM-Situationsanalyse** — Regime-Erkennung, Kontextsignale, taktische Parametersteuerung
2. **Deterministische Regellogik** — Entry/Exit-Entscheidungen, Pattern-State-Machines
3. **Quantitative Validierung** — Indikatoren, Scoring, Veto-Mechanismus

### Kernprinzipien

| Prinzip | Bedeutung |
|---------|----------|
| **Asymmetrische Verantwortung** | Das LLM analysiert und parametrisiert — deterministische Regeln entscheiden und handeln |
| **Simulationsfaehigkeit als First-Class-Concern** | Live und Simulation durchlaufen denselben Codepfad. Unterschiede nur ueber austauschbare Adapter und die Clock |
| **EOD-Flat** | Am Ende jedes Handelstages MUESSEN alle Positionen geschlossen sein |
| **Determinismus** | Gleicher Input fuehrt zum gleichen Ergebnis. Keine Zufallskomponente |
| **MarketClock** | Einzige Zeitquelle im Trading-Codepfad. Kein `Instant.now()` |

### Systemparameter

| Parameter | Wert |
|-----------|------|
| Tageskapital | 10.000 EUR (aufgeteilt auf aktive Pipelines) |
| Max. Tagesverlust (Hard-Stop) | -10% (1.000 EUR) — global ueber alle Pipelines |
| Instrumente | 1–3 parallel (je Pipeline ein Instrument) |
| Richtung | Long only |
| Instrumenttyp | Aktien |
| Handelszeiten | Pre-Market + RTH |
| Autonomie | Vollautomatisch |
| Max. Zyklen pro Tag pro Pipeline | 3 (konfigurierbar) |

### Vier-Schichten-Architektur

1. **Datenschicht** — Marktdaten empfangen, validieren, puffern (odin-data)
2. **Entscheidungsschicht (Brain)** — LLM Analyst, KPI Engine, Rules Engine, Quant Validation, Decision Arbiter (odin-brain)
3. **Risiko- und Ausfuehrungsschicht** — Risk Gate, OMS, Broker-Anbindung (odin-execution, odin-broker)
4. **Praesentation & Monitoring** — SSE Gateway, REST Controls, React Dashboard (odin-frontend, odin-app)

### Fuenf zentrale Ports

| Port | Live-Implementierung | Simulation-Implementierung |
|------|---------------------|---------------------------|
| MarketClock | SystemMarketClock (Exchange-TZ) | SimClock (vom Runner gesteuert) |
| MarketDataFeed | IbMarketDataFeed (TWS API) | HistoricalMarketDataFeed (Replay) |
| BrokerGateway | IbBrokerGateway (TWS API) | BrokerSimulator (Fill-Modell) |
| LlmAnalyst | ClaudeAnalystClient / OpenAiAnalystClient | CachedAnalyst (aufgezeichnete Responses) |
| EventLog | PostgresEventLog (async) | PostgresEventLog (synchron in Barrier) |

> **Detaillierte Architektur:** ARCH-00 (Systemuebersicht), ARCH-01 (Modularchitektur), ARCH-02 bis ARCH-10 (Detailkapitel)

---

## 2. Strategielogik

### 2.1 Tages-Lifecycle

> **Zeitzone:** Alle Uhrzeiten in **US Eastern Time (ET)**. Implementierung MUSS Sommerzeitwechsel korrekt behandeln (Exchange-TZ `America/New_York`).

| Phase | Zeitraum | Verhalten |
|-------|----------|-----------|
| Pre-Market | 07:00 – 09:30 ET | System-Initialisierung, LLM-Tagesplan, passive Beobachtung. Trades nur bei aktiviertem Pre-Market-Trading (halbe Position, verschaerfte Regeln) |
| Opening Auction | 09:30 – 09:45 ET | Reine Diagnose. Kein Trade. ATR und Volumen-Baseline kalibrieren |
| Aktive Handelsphase | 09:45 – 15:15 ET | Decision Loop laeuft. Entries und Exits erlaubt |
| Power Hour | 15:15 – 15:45 ET | Verschaerfte Regeln. **Entries bis 15:30 erlaubt**, danach keine neuen Positionen. Enge Trailing-Stops |
| Forced Close | 15:45 – 15:55 ET | Alle offenen Positionen werden zwangsgeschlossen |
| End-of-Day | 15:55 – 16:15 ET | Reconciliation, Performance-Logging |

### 2.2 Pipeline-Zustandsmaschine (FSM)

| State | Beschreibung | LLM aktiv? | Trading erlaubt? |
|-------|-------------|------------|-----------------|
| INITIALIZING | Systemstart, Daten laden | Ja (Planung) | Nein |
| WARMUP | Indikator-Warmup, Decision Loop laeuft mit Guards | Ja (Analyse) | Nein (Entry blockiert) |
| OBSERVING | Markt beobachten, Regime diagnostizieren | Ja (Analyse) | Nein |
| SEEKING_ENTRY | Aktiv nach Entry-Punkt suchen | Ja (Kontext) | Ja (Entry) |
| PENDING_FILL | Order gesendet, warte auf Fill | Nein | Nein (wartend) |
| MANAGING_TRADE | Position offen, Stop aktiv, Exit-Timing | Ja (Kontext) | Ja (Exit/Adjust) |
| **FLAT_INTRADAY** | Position komplett geschlossen, Tag laeuft weiter | Ja (Analyse) | Ja (Re-Entry moeglich) |
| DAY_STOPPED | Tagesverlust-Limit erreicht oder Max-Cycles | Nein | Nein |
| FORCED_CLOSE | Handelszeit vorbei, Positionen schliessen | Nein | Nur Exits |
| EOD | Reconciliation und Logging | Nein | Nein |

**FLAT_INTRADAY** ist ein neuer State (Stakeholder-Entscheidung, **Status: Geplant — Roadmap P4a**), der den Multi-Cycle-Day ermoeglicht. Er unterscheidet sich von OBSERVING durch den mitgefuehrten Zyklen-Kontext (realisierter P&L, verbrauchtes Risk-Budget, Cycle-Counter). Bis zur Implementierung wechselt die Pipeline nach komplettem Exit in OBSERVING.

**State-Uebergaenge:**

```
INITIALIZING → WARMUP → OBSERVING ↔ SEEKING_ENTRY → PENDING_FILL → MANAGING_TRADE
                                                                           │
                                                                    FLAT_INTRADAY
                                                                           │
                                                              → SEEKING_ENTRY (Re-Entry)
                                                              → FORCED_CLOSE
                                                              → DAY_STOPPED
```

### 2.3 Decision Loop

Der Decision Loop orchestriert alle Entscheidungsschritte pro Decision-Bar (1-Minuten-Bar-Close):

1. **Hard-Stop-Pruefung** — Tagesverlust >= 10%? → DAY_STOPPED
2. **MarketSnapshot** erzeugen (immutable, MarketClock-Timestamp)
3. **KPI-Engine** — Indikatoren berechnen → IndicatorResult
4. **LlmAnalysis** aus LlmAnalysisStore lesen (async bereitgestellt)
5. **Regime-Bestimmung** (KPI primaer, LLM sekundaer)
6. **Rules Engine** — Entry/Exit-Bedingungen pruefen
7. **Quant Validation** — Score + Hard-Veto
8. **Decision Arbiter** — Finale Entscheidung
9. **Risk Gate** — Position Sizing, Limits, R/R
10. **OMS** — Order erzeugen oder Intent verwerfen + EventLog

> **Ein-Intent-pro-Bar:** Maximal ein TradeIntent pro Decision-Cycle. Tie-Break: Exit > Entry. **Trailing-Stops und Scaling-Out sind OMS-Maintenance**, keine TradeIntents.

### 2.4 Regime Detection

Die Regime-Bestimmung folgt dem Prinzip **KPI primaer, LLM sekundaer**:

| Regime | KPI-Kriterien |
|--------|--------------|
| TREND_UP | EMA(9) > EMA(21) AND Preis > VWAP AND ADX > 20 (konfigurierbar) |
| TREND_DOWN | EMA(9) < EMA(21) AND Preis < VWAP AND ADX > 20 (konfigurierbar) |
| RANGE_BOUND | ADX < 20 AND Preis innerhalb VWAP ± 1.0x ATR (konfigurierbar) |
| HIGH_VOLATILITY | ATR(aktuell) > 1.5 x ATR(20-Bar-SMA) (konfigurierbar) |
| UNCERTAIN | Keines der obigen klar erfuellt |

**Bei KPI/LLM-Widerspruch:** Das konservativere Regime gilt (im Long-Only-Kontext: UNCERTAIN > TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > TREND_UP).

**Regime-Wechsel:** Erst nach zwei aufeinanderfolgenden Bar-Closes mit konsistentem neuem Regime.

### 2.5 Entry-Regeln

**Vorbedingung (Mandatory Gate):** Ein frischer LLM-Output mit `regimeConfidence > 0` MUSS vorliegen. Ohne frischen LLM-Output ist `regimeConfidence = 0.0` und Entries sind blockiert. Das LLM **triggert** keinen Entry — es liefert ein notwendiges Input-Feature.

| Regime-Confidence | Quant-Score-Schwelle |
|-------------------|---------------------|
| < 0.5 | Kein Entry (Regime = UNCERTAIN) |
| 0.5 – 0.7 | >= 0.70 (erhoehte Schwelle) |
| > 0.7 | >= 0.50 (Standard-Schwelle) |

| Regime | Entry-Bedingung |
|--------|----------------|
| TREND_UP | EMA(9) > EMA(21) AND RSI < 70 AND Preis <= VWAP + 0.5x ATR(14) |
| TREND_DOWN | **Default: Kein Entry.** Aggressive Mode (Config OFF): RSI < 25, Volume > 2x SMA, halbe Position |
| RANGE_BOUND | RSI < 40 AND Preis nahe VWAP-Support |
| HIGH_VOLATILITY | Volume > 1.5x SMA AND Spread < 0.3% |
| UNCERTAIN | **Kein Entry** |

### 2.6 Exit-Regeln (Prioritaetsordnung)

Exit-Trigger werden in fester Rangfolge geprueft. Hoehere Prioritaet uebersteuert niedrigere:

| Prio | Trigger | LLM-Abhaengigkeit |
|------|---------|-------------------|
| 1 | **Stop-Loss** (Preis <= Entry - 2.2 x entryAtr) | Keine (Broker-seitig, GTC) |
| 2 | **Forced Close** (MarketClock > RTH-Ende - Buffer) | Keine |
| 3 | **Trailing-Stop** (Preis faellt um Trail-Factor x ATR vom High) | Keine |
| 4 | **Regime-Wechsel** (2x bestaetigt, Position passt nicht) | Wuenschenswert |
| 5 | **TACTICAL_EXIT** (LLM exit_bias + KPI-Bestaetigung) | Nur als Signal |
| 6 | **Zeitbasiert** (hold_duration_bars ueberschritten) | Kontext |
| 7 | **Gewinnmitnahmen** (R-basierte Tranchen) | Keine |

**TACTICAL_EXIT** (Stakeholder-Entscheidung) ist eine neue Exit-Prioritaet zwischen REGIME_CHANGE und zeitbasiertem Exit. Das LLM allein erzwingt KEINEN Exit — mindestens ein KPI-Signal MUSS bestaetigen.

### 2.7 Pattern-State-Machines

Die Rules Engine kennt zwei konkrete Intraday-Setups:

**Setup A: Flush → Reclaim → Run (V-Reversal)**

| State | Uebergang |
|-------|-----------|
| FLUSH_DETECTED | Bar-Range > 2x ATR, Close nahe Low → RECLAIM wenn Preis innerhalb 3 Bars > VWAP |
| RECLAIM | Preis ueber VWAP + MA-Cluster, Hold 2 Bars → RUN bei Higher-Low. **Starter-Entry** |
| RUN | Hoehere Tiefs, flache Pullbacks → EXIT bei Lower-Low. **Add-Position bei Bestaetigung** |
| EXIT | Strukturbruch → IDLE |

**Setup B: Coil → Breakout (Volatilitaets-Kontraktion)**

| State | Uebergang |
|-------|-----------|
| COIL_FORMING | Fallende Hochs + steigende Tiefs, min. 5 Bars → COIL_MATURE bei > 40% Kontraktion |
| COIL_MATURE | Range < 60% Initial, nahe Apex → BREAKOUT bei Close ausserhalb + Range > 1.5x ATR |
| BREAKOUT/RUN | Trend etabliert → EXIT bei Reentry in Formation |

**Pattern-Aktivierung:** LLM schlaegt Patterns vor (`patternCandidates`), aber State-Machine-Uebergaenge werden ausschliesslich durch KPI-Kriterien getriggert. Pattern-Confidence >= 0.5 UND deterministische KPI-Bestaetigung MUESSEN beide vorliegen.

> **Timeframe:** Pattern-State-Machines operieren auf **1-Minuten-Bars** (Decision-Bar-Ebene). ATR-Referenzen in Pattern-Definitionen (z.B. "Range > 2x ATR") verwenden ATR(14) auf 5-Min-Bars. Diese Mischung ist beabsichtigt: feinere Granularitaet fuer Pattern-Erkennung, robustere ATR fuer Normalisierung.

### 2.8 Pre-Market-Trading

Pre-Market-Trading ist per Konfigurationsschalter steuerbar (`odin.core.pipeline.pre-market-trading-enabled`, **Default: false**). Bei aktiviertem Schalter darf der Agent waehrend der Pre-Market-Phase von OBSERVING nach SEEKING_ENTRY wechseln. Es gelten verschaerfte Regeln:

| Regel | Wert |
|-------|------|
| Max. Positionsgroesse | 25% Tageskapital (halbe Standardgroesse) |
| Mindest-Spread | < 0.5% (Bid-Ask relativ zum Midpoint) |
| Mindest-Volumen | > 10.000 Stueck in den letzten 30 Min Pre-Market |
| Repricing | Kein Repricing — ein Limit-Order-Versuch (Override der Standard-Execution-Policy MC-6.6) |

### 2.9 Edge Cases und Market Exceptions

| Szenario | Erkennung | Reaktion |
|----------|-----------|---------|
| **Trading Halt / LULD Pause** | Feed zeigt Halt | Agent pausiert. Nach Resume: LLM-Re-Assessment bevor neue Aktion |
| **LULD ueber Close** | Halt dauert bis nach 15:55 ET | Position kann nicht geschlossen werden. Broker-seitige MOC-Order als Fallback. Am naechsten Tag Reconciliation |
| **Flash Crash** (> 5% in < 1 Min) | Crash-Detection (DQ-Gate) | Sofortiger Exit, DAY_STOPPED |
| **Cannot Liquidate** | Broker lehnt Close-Order ab | Eskalation: Alert an Operator, Retry alle 30s, manueller Eingriff dokumentieren |
| **Instrument Halt vor RTH** | Trading-Restriction auf zugewiesenem Instrument | Pipeline wechselt in DAY_STOPPED (kein Ersatzsymbol, andere Pipelines laufen weiter) |
| **Abnormale Fill-Rate** (< 30%) | Ueber 5 Versuche gemessen | Positionsgroesse halbieren oder OBSERVING |
| **Quote Staleness** (> 15s) | Best Bid/Ask aelter als 15s bei aktivem Markt (NBBO-basiert) | Order-Sperre bis frische Quotes vorliegen. Ab 30s: DQ-Gate-Warnung. Ab 60s: Kill-Switch via DQ-Gate (Stale Quote Detection) |
| **Reclaim ohne Follow-through** (Bull Trap) | Setup A: Preis reclaimed, wird sofort abverkauft | Initial-Stop schuetzt. Kein Re-Entry fuer dieses Pattern heute |
| **Fakeout am Apex** | Setup B: Breakout wird negiert | Exit bei Reentry in Formation. Am Apex kein Entry |

### 2.10 Instrument-Selektion

Die Instrumente werden **extern vorgegeben** und beim Start der Pre-Market-Phase (07:00 ET) eingefroren. Kein Symbolwechsel waehrend des Handelstages. Die Auswahl erfolgt ausserhalb des Agenten (manuell oder durch ein vorgelagertes Screening). ODIN handelt nur vorgegebene Symbole — es gibt kein internes Instrument-Discovery.

**Anforderungen an geeignete Instrumente:**
- Aktien (US, Europa)
- Ausreichende Liquiditaet (Spread < 0.5%, taegl. Volumen > 500.000)
- Keine Penny Stocks (Preis > 5 USD/EUR)
- Handelbar ueber IB TWS

### 2.11 Position Building: Starter + Add

Fuer Setup A wird eine gestufte Positionsaufbau-Strategie verwendet:

| Phase | Anteil | Zeitpunkt | Stop |
|-------|--------|-----------|------|
| Starter | ~40% der Zielgroesse | RECLAIM-State | Unter Flush-Low |
| Add | ~60% der Zielgroesse | RUN-State bestaetigt | Unter letztes Higher-Low |

Gesamtrisiko (Starter + Add) DARF den max_risk pro Trade (3% Tageskapital) NICHT ueberschreiten.

> **Details:** ARCH-06 (Rules Engine), ARCH-07 (OMS)

---

## 3. Multi-Cycle-Day

### 3.1 Paradigmenwechsel (Stakeholder-Entscheidung)

ODIN unterstuetzt **mehrere vollstaendige Entry/Exit-Zyklen pro Tag** auf demselben Instrument. Dies ist DER zentrale Use-Case fuer die LLM-Integration — ein statischer Algorithmus kann nicht zuverlaessig entscheiden, ob ein Plateau temporaer oder terminal ist.

### 3.2 FLAT_INTRADAY State (Stakeholder-Entscheidung)

Ein neuer Pipeline-State FLAT_INTRADAY MUSS nach komplettem Exit eingefuehrt werden. Er traegt den Kontext des abgeschlossenen Zyklus:

- Realisierter P&L
- Verbrauchtes Risk-Budget
- Cycle-Counter

**State-Uebergaenge:**

```
MANAGING_TRADE → FLAT_INTRADAY     (komplett verkauft, Handelstag laeuft)
FLAT_INTRADAY → SEEKING_ENTRY      (LLM: ALLOW_RE_ENTRY + KPI-Bestaetigung)
FLAT_INTRADAY → FORCED_CLOSE       (Handelszeit endet)
FLAT_INTRADAY → DAY_STOPPED        (Hard-Stop oder Max-Cycles)
```

### 3.3 Zyklen-Typen

| Zyklus-Typ | Timing | Positionsgroesse | Besonderheit |
|------------|--------|-----------------|--------------|
| **Trend-Riding** (Zyklus 1) | Fruehe RTH | Standard | Proaktiver Exit beim Plateau via TACTICAL_EXIT |
| **Recovery-Trade** (Zyklus 2+) | Nach Korrektur (typ. 30-90 Min) | Reduziert (Default 60%) | Konservativere Entry-Schwellen, engerer initialer Stop |

### 3.4 Re-Entry vs. Scale-Up (Aufstocken)

| Merkmal | Re-Entry | Aufstocken (Scale-Up) |
|---------|----------|----------------------|
| Position zwischen Zyklen | 0 (komplett flat) | > 0 (Runner gehalten) |
| Pipeline-State | FLAT_INTRADAY | MANAGING_TRADE (unveraendert) |
| Trigger | `entry_timing_bias = ALLOW_RE_ENTRY` | LLM-Recovery-Signal + KPI bei bestehender Position |
| Semantik | Neuer Trade | Vergroesserung einer bestehenden Position |

**Scale-Up OMS-Implikationen:**
- Aufgestockter Teil hat eigenen Entry-Preis und Entry-ATR
- Position besteht aus Tranchen mit unterschiedlichen Entry-Parametern
- P&L-Berechnung pro Tranche
- Cycle-Counter zaehlt Aufstocken als eigenen Zyklus

### 3.5 Cycle-Counter (Stakeholder-Entscheidung)

Pro Pipeline MUSS ein `cycleNumber` gefuehrt werden. Maximum: 3 (konfigurierbar, Hard-Cap als Guardrail).

### 3.6 Guardrails fuer Multi-Cycle

| Guardrail | Regel |
|-----------|-------|
| Max-Cycles-Per-Day | 3 (konfigurierbar) |
| Cooling-Off nach Exit | Min. 15 Minuten zwischen Zyklen |
| Profit-Gate | Re-Entry nur wenn vorheriger Zyklus profitabel |
| Budget-Gate | Verbleibendes Risk-Budget MUSS minimale Position erlauben |
| LLM-Pflicht | Re-Entry nur mit frischem LLM-Assessment (`ALLOW_RE_ENTRY`) |
| Zeitfenster | Kein Re-Entry nach 15:15 ET (30 Min vor FORCED_CLOSE). **Strenger als Erst-Entry** (bis 15:30 erlaubt) |

### 3.7 LLM-Rolle bei Multi-Cycle

Das LLM MUSS vier Arten von Entscheidungen unterstuetzen:
1. **Plateau-Erkennung** — Exhaustion-Signale fuer proaktiven Exit (via TACTICAL_EXIT)
2. **Drawdown-Timing** — Bewertung ob Korrektur abgeschlossen ist
3. **Recovery-Signal** — Identifikation nachhaltiger Erholung
4. **Re-Entry-Freigabe** — `entry_timing_bias = ALLOW_RE_ENTRY`

> **Details:** Strategie-Optimierung-Sparring Teil 5

---

## 4. Trailing Stop und Profit Protection

### 4.1 Grundprinzipien (Stakeholder-Entscheidungen)

| Prinzip | Regelung | Status |
|---------|----------|--------|
| **Highwater-Mark** | Trailing Stop DARF nur steigen, nie fallen. `effectiveStop = max(prevStop, stop0, trailCandidate, mfeLockStop)` | Implementiert |
| **ATR-Freeze** | ATR wird bei Entry eingefroren. Faktor 2.2 fuer Stop-Berechnung, R-Normalisierung, Trail-Basis | Implementiert |
| **Runner-Trail** | Der bisherige Runner-Trail (EMA(9)/Higher-Low) bleibt als zusaetzlicher Stop-Kandidat in `max()` bestehen | Bestaetigt |
| **Profit Protection als Floor** | R-basierte Tabelle definiert Mindest-Stop-Level. LLM-Parameter KOENNEN innerhalb des Floors variieren, aber nie darunter | Bestaetigt |

### 4.2 R-basierte Profit-Protection Stufentabelle

| R-Schwelle | Stop-Typ | Trail-ATR-Factor | MFE-Lock % |
|------------|----------|-----------------|------------|
| < 1.0R | Nur initialer Stop (`stop0`) | — | — |
| >= 1.0R | Break-Even: Stop >= Entry + 0.10R (Puffer fuer Fees) | — | — |
| >= 1.5R | Trail aktiv (wide) | 2.75 | — |
| >= 2.0R | Trail enger + MFE-Lock | 2.00 | 35% |
| >= 3.0R | Aggressiver Trail + Lock | 1.25 | 60% |
| >= 4.0R | Final Protect | 1.00 | 75% |

**MFE-Lock:** Sichert einen Mindestprozentsatz des bisher erreichten Maximum Favorable Excursion (MFE) ab.

**Berechnung:**
```
effectiveStop = max(prevStop, stop0, trailCandidate, mfeLockStop)
```

### 4.3 Regime-spezifische Trail-Weite

| Regime | Trail-Faktor `k` |
|--------|------------------|
| TREND_UP stark (ADX hoch) | 2.0 – 3.0 (breiter, Trend Luft lassen) |
| TREND_UP schwach / Chop | 1.2 – 2.0 (enger) |
| HIGH_VOLATILITY | Initial 3.0, dann via Profit-Protection progressiv enger |

### 4.4 Runner-Trailing-Regeln

| Situation | Trailing-Regel |
|-----------|---------------|
| Standard | Trail unter EMA(9) oder letztes Higher-Low — je nachdem was naeher am Kurs |
| Ueberhitzt (RSI > 80, oberes Bollinger-Band) | Engerer Trail, unter letzte Kerze |
| Ruecksetzer gesund (Preis haelt MA-Zone) | Runner halten |
| Ruecksetzer kippt (Close unter MA-Zone, Lower-Low) | Runner sofort schliessen |

### 4.5 Precedence-Kette und Konfliktaufloesung (Stakeholder-Entscheidung)

Die strikte Rangfolge aller Einflussquellen:

1. **Hard-Exits** (KillSwitch, ForcedClose, StopLoss, TrailingStop) — IMMER zuerst, LLM-unabhaengig
2. **Profit-Protection** setzt Floor der Tightness
3. **LLM `trail_mode`** darf innerhalb der Grenzen variieren, aber nie gegen (2)
4. **LLM `exit_bias`** kann nur Soft-Exit-Schwellen veraendern, nicht Hard-Exits

**Konfliktaufloesung:**
```
effectiveTrailFactor = min(factor_from_llm, factor_from_profit_protection)
```

Es gilt IMMER der engere (konservativere) Wert.

> **Details:** ARCH-07 (OMS), Strategie-Optimierung-Sparring Teil 1+2

---

## 5. LLM-Integration

### 5.1 Architekturprinzip: Tactical Parameter Controller (Stakeholder-Entscheidung)

Das LLM ist KEIN reiner Analyst mehr — es ist ein **Tactical Parameter Controller**. Es entscheidet nicht BUY/SELL, sondern liefert **gebundene, strukturierte Parameter** (bounded Enums), die deterministisch in Regeln uebersetzt werden.

> **Eiserne Regel:** Kein LLM-Output DARF direkt einen Trigger ausloesen. LLM-Features fliessen in die Rules Engine, die eigenstaendig und deterministisch entscheidet.

### 5.2 Provider-Anbindung (Stakeholder-Entscheidung)

| Provider | SDK/Protokoll | Rolle |
|----------|--------------|-------|
| **Claude** | **Claude Agent SDK** (Java) | Primaer. Strukturierte Analyse via Tool-Use/Structured-Output |
| **OpenAI** | OpenAI API (REST, pay-per-use) | Alternativ. A/B-Tests, Evaluation. **Kein Runtime-Failover** |

Der aktive Provider wird **vor Handelsstart per Konfiguration** gewaehlt und steht fuer den gesamten Handelstag fest. Bei Ausfall greift der Circuit Breaker → Quant-Only-Modus, KEIN automatischer Provider-Wechsel.

**Backtest LLM Provider** (Stakeholder-Entscheidung): Konfigurierbar CACHED / CLAUDE / OPENAI.

### 5.3 LLM-Analyse-Schema (Decision-Features)

| Kategorie | Felder | Nutzung |
|-----------|--------|---------|
| **Decision-Features** | `regime`, `regime_confidence`, `pattern_candidates` | Input fuer Rules Engine und Quant Validation |
| **Kontext-Features** | `opportunity_zones`, `entry_price_zone`, `target_price`, `hold_duration_bars`, `urgency_level` | Optionale Kontextsignale fuer Rules Engine |
| **Logging-Only** | `reasoning`, `key_observations`, `risk_factors`, `market_context_signals` | Nur Telemetrie/Logging, NIE in Entscheidungslogik |

### 5.4 Tactical Decision-Features (Stakeholder-Entscheidung)

**P0 — Kern (bounded Enums):**

| Feature | Typ | Interpretation |
|---------|-----|---------------|
| `exit_bias` | Enum: HOLD, NEUTRAL, EXIT_SOON, EXIT_NOW | Aktiviert TACTICAL_EXIT-Check (siehe MC-5.5) |
| `trail_mode` | Enum: WIDE, NORMAL, TIGHT | Multiplikator auf base trailingStopAtrFactor: WIDE=1.5x, NORMAL=1.0x, TIGHT=0.75x. Clamp auf [minTrail, maxTrail] |
| `profit_protection_profile` | Enum: OFF, STANDARD, AGGRESSIVE | OFF = keine Profit-Protection. STANDARD = R-basierte Stufentabelle (MC-4.2) ab 1.5R. AGGRESSIVE = Stufentabelle ab 1.0R mit engeren Faktoren |
| `target_policy` | Enum: KEEP, CAP_AT_R_MULTIPLE, TRAIL_ONLY | KEEP = Standard-Targets. CAP_AT_R_MULTIPLE = Target deckeln bei aktuellem R-Multiple. TRAIL_ONLY = alle Targets stornieren, nur Trail (MC-6.4) |

**P1 — Regime-Feinsteuerung:**

| Feature | Typ | Interpretation |
|---------|-----|---------------|
| `subregime` | Enum pro Regime | Beeinflusst Entry-Strenge und Exit-Defaults |
| `exhaustion_signal` | Enum: NONE, EARLY_WARNING, CONFIRMED | Schaltet Protect-Policy |

**P2/P3 — Spaeter:**

| Feature | Typ |
|---------|-----|
| `entry_timing_bias` | Enum: ALLOW_NOW, WAIT_PULLBACK, WAIT_CONFIRMATION, ALLOW_RE_ENTRY, SKIP |
| `scale_out_profile` | Enum: OFF, CONSERVATIVE, STANDARD, AGGRESSIVE |

### 5.5 TACTICAL_EXIT (Stakeholder-Entscheidung)

TACTICAL_EXIT ist eine neue Exit-Prioritaet zwischen REGIME_CHANGE (Prio 4) und zeitbasiertem Exit (Prio 6). Es ist der primaere Mechanismus fuer LLM-Einfluss auf Exit-Timing.

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
| NEUTRAL / HOLD | TACTICAL_EXIT deaktiviert |
| EXIT_SOON | Exit wenn mind. 2 von 4 Signalen true ODER 1 Signal + exhaustion >= EARLY_WARNING |
| EXIT_NOW | Exit wenn mind. 1 Signal true UND (exhaustion == CONFIRMED ODER MFE >= 1R) |

> **Eiserne Regel:** Das LLM allein erzwingt KEINEN Exit. Ohne mindestens ein bestaetigendes KPI-Signal bleibt die Position offen.

### 5.6 TRAIL_ONLY als OMS-Override (Stakeholder-Entscheidung)

Wenn `target_policy = TRAIL_ONLY`, MUSS das OMS alle Target-Orders ignorieren und ausschliesslich den Trailing-Stop verwenden. Dies ist ein LLM-gesteuerter Override, der nur den Runner-Anteil oder die gesamte Position betreffen kann (konfigurierbar).

### 5.7 Subregimes

Das 5-stufige Regime-Modell wird um Subregimes erweitert:

| Regime | Subregimes |
|--------|-----------|
| TREND_UP | EARLY, MATURE, LATE, EXHAUSTION |
| TREND_DOWN | RELIEF_RALLY, PULLBACK, ACCELERATION, CAPITULATION |
| RANGE_BOUND | MEAN_REVERT, RANGE_EXPANSION, BREAKOUT_ATTEMPT, CHOP |
| HIGH_VOLATILITY | NEWS_SPIKE, MOMENTUM_EXPANSION, AFTERSHOCK_CHOP, LIQUID_VOL |
| UNCERTAIN | TRANSITION, MIXED_SIGNALS, LOW_LIQUIDITY, DATA_QUALITY |

**Bestimmung:** KPI-Engine primaer, LLM als sekundaerer Hint. Bei Widerspruch: konservativeres Subregime.

### 5.8 Exhaustion-Detection (KPI-basiert)

Drei Saeulen, ALLE muessen mindestens ein true-Kriterium liefern:

**Saeule A — Extension:** RSI >= 78 ODER Bollinger-Extension ODER VWAP-Extension
**Saeule B — Climax:** volumeRatio >= 2.5 ODER Bar-Range/ATR >= 1.8
**Saeule C — Rejection:** BB Re-Entry ODER VWAP Snapback ODER Fast RSI Drop ODER Trend Stall

```
exhaustion_kpi_confirmed = A AND B AND C
```

### 5.9 Anti-LLM-Drift Guardrails

| Guardrail | Regel |
|-----------|-------|
| R-abhaengige Mindestweiten | Vor 1.5R ist Trail verboten. 1.5-2R: mindestens 2.5 als Factor |
| Mode-Hysterese | `trail_mode` darf nur wechseln bei 2 consecutive Bars mit gleichem Mode |
| Monitoring | pct_tight > 80% UND median_R_captured < 0.35R → LLM-Influence zurueck auf NORMAL |

### 5.10 Aufruf-Strategie

| Volatilitaet | Periodisches LLM-Intervall |
|--------------|---------------------------|
| Hoch (> 120% Daily-ATR) | Alle 3 Minuten |
| Normal (80-120%) | Alle 5 Minuten |
| Niedrig (< 80%) | Alle 15 Minuten |

**Event-Trigger (zusaetzlich zum periodischen Intervall):**

| Trigger | Bedingung |
|---------|-----------|
| Entry-Approaching | Preis betritt eine Opportunity-Zone (Edge-triggered) oder naehert sich `entry_price_zone` auf < 0.5x ATR(14). Debounce: 90s zwischen proaktiven Refreshes |
| Signifikante Ereignisse | VWAP-Durchbruch, Volumen-Spike > 2x, Stop-Loss-Annaeherung, Profit-Target erreicht |
| State-Transition | Jeder Zustandswechsel der FSM |

> **Freshness-Garantie bei Low-Vol:** Das periodische 15-Min-Intervall allein wuerde die Entry-Freshness-Anforderung (< 120s) verletzen. Die Event-Trigger (insbesondere Entry-Approaching) sorgen dafuer, dass bei einer tatsaechlichen Entry-Evaluation IMMER ein frischer LLM-Output vorliegt. Kein Entry ohne frischen Kontext — auch nicht bei 15-Min-Grundintervall.

**Single-Flight:** Nie mehr als ein laufender LLM-Call gleichzeitig pro Pipeline. Weitere Trigger werden koalesziert.

### 5.11 Safety Contract

5 Schichten:
1. **Schema-Validierung** — Striktes JSON-Schema, keine Toleranz
2. **Plausibilitaetspruefung** — ATR-relative Schwellen
3. **Konsistenzpruefung** — Feld-Kombinationen
4. **TTL/Freshness** — MarketClock-basiert (Entry: < 120s, Management: < 900s)
5. **Historische Kalibrierung** — Ab 100 Handelstagen Confidence-Pruefung

### 5.12 News Security und Input Validation

Externe Textdaten (Nachrichtenartikel, Social-Media, Analystenkommentare) sind potentielle Angriffsvektoren fuer Prompt Injection. Alle externen Texte werden als **untrusted input** behandelt:

| Massnahme | Detail |
|-----------|--------|
| Whitelist-Felder | Nur strukturierte Felder ans LLM: Headline (max 200 Chars), Source, Timestamp, Sentiment-Score |
| Keine Rohtexte | Artikel-Body wird NIEMALS in den LLM-Prompt eingespeist |
| Character-Sanitization | Nur ASCII + Standard-Unicode. Control Characters, Zero-Width-Zeichen werden entfernt |
| Length Limits | Jedes String-Feld hat striktes Max-Length mit Truncation |
| Prompt-Injection-Schutz | Externe Texte in dediziertem `external_context` JSON-Block, nicht als System/User-Prompt |
| Anomalie-Detection | LLM-Output nach News-Einspeisung signifikant abweichend → verwerfen und ohne News wiederholen |

### 5.13 Circuit Breaker und Quant-Only-Modus

| Stufe | Schwelle | Reaktion |
|-------|----------|----------|
| Per-Call-Timeout | Response > 10s | Call abbrechen |
| Quant-Only | 3 Failures in Folge | Keine neuen Entries, bestehende Positionen trailen |
| Trade-Halt | > 30 Min ohne Erfolg | Nur passive Verwaltung |

> **Details:** ARCH-05 (LLM-Integration)

---

## 6. OMS

### 6.1 Scaling-Out-Strategie (Multi-Tranchen)

Die Tranchenanzahl ist **budgetabhaengig**:

| Variante | Bedingung | Tranchen | Aufteilung |
|----------|----------|----------|------------|
| Kompakt (3T) | < 5.000 USD | 3 | 40% @ 1R, 40% @ 2R, 20% Runner |
| Standard (4T) | 5.000–15.000 USD | 4 | 30% @ 1R, 30% @ Key-Level, 25% @ 2R, 15% Runner |
| Voll (5T) | > 15.000 USD | 5 | 25% @ 1R, 25% @ Key-Level, 25% @ 2R, 15% @ 3R, 10% Runner |

**Schwellenwerte und Prozentaufteilungen sind als Properties konfigurierbar.** Die USD-Schwellenwerte gelten fuer US-Aktien. Fuer EU-Aktien werden die Schwellenwerte in EUR mit dem aktuellen EUR/USD-Spot umgerechnet.

### 6.2 Order-Architektur

- **Stop-Loss-Order:** Eigenstaendige Stop-Order auf **gesamte Restposition**. NICHT in OCA-Gruppe. GTC beim Broker (ueberlebt Crash). Wird bei jedem Teil-Exit angepasst
- **Target-Orders:** Separate Limit-Sell-Order pro Tranche. Targets untereinander unabhaengig (keine OCA)
- **Kein OCA fuer Stop/Target-Kombination** — Synchronisation ueber interne OMS-Logik

### 6.3 Stop-Nachfuehrung

Die Stop-Nachfuehrung erfolgt auf **Tick-Ebene** (nicht nur bei Bar-Close). Sie ist OMS-Maintenance, kein TradeIntent.

Bei Tranche 1 Fill: Stop auf Break-Even (Entry + 0.10R).

### 6.4 TRAIL_ONLY Override (Stakeholder-Entscheidung)

Wenn `target_policy = TRAIL_ONLY` vom LLM geliefert wird, MUSS das OMS:
- Alle offenen Target-Orders stornieren
- Ausschliesslich den Trailing-Stop als Exit-Mechanismus verwenden
- Die gesamte Restposition per Trail managen

> **Nebenwirkung:** TRAIL_ONLY deaktiviert implizit das Scaling-Out ueber Target-Orders. Die gesamte Restposition wird ausschliesslich per Trailing-Stop gemanagt. Dies ist beabsichtigt fuer Trend-Days, bei denen partielle Gewinnmitnahmen den Gesamtprofit schmaelern wuerden.

### 6.5 Multi-Cycle-Handling

| Aspekt | Regelung |
|--------|---------|
| Cycle-Counter | Pro Pipeline, `cycleNumber` in Events |
| Unabhaengige Trades | Jeder Zyklus = eigenstaendiger Trade mit eigenen Stops, Targets, P&L |
| Positionsgroessen | Zyklus 2+: Sizing beruecksichtigt verbrauchtes Tagesbudget |

### 6.6 Execution Policy

| Situation | Order-Typ |
|-----------|----------|
| Normal Entry | Limit Order, Repricing alle 5s (max. 3 Zyklen) |
| Normal Exit (Tranchen) | Limit Order, Repricing alle 5s |
| Forced Close (15:45+) | Market Order |
| Kill-Switch | Market Order |
| Stop-Loss | Stop-Limit (Offset 0.1%), Fallback: Stop-Market nach 5s |

**Market Orders sind AUSSCHLIESSLICH erlaubt bei:** Forced Close, Kill-Switch, Stop-Market-Fallback, Partial-Fill-Notfall, Gap-Opening (= Markteroeffnung bei der der Preis unter dem Stop-Level liegt, z.B. nach Overnight-Gap oder LULD-Resume).

**Repricing-Degradation:** Bei hoher Cancel/Replace-Rate MUSS das Repricing-Verhalten degradiert werden:

| Bedingung | Reaktion |
|-----------|----------|
| Cancel/Replace-Rate > 5:1 (ueber 30 Min) | Repricing-Intervall von 5s auf 10s erhoehen |
| Order/Trade-Ratio > 10:1 (ueber 60 Min) | Repricing auf 1 Zyklus (statt 3), kein Abandon-Retry |
| Fill-Rate < 30% (ueber 5 Versuche) | Positionsgroesse halbieren ODER in OBSERVING |

### 6.7 Fill-Event-Handling

| Event | OMS-Reaktion |
|-------|-------------|
| Target-Tranche gefuellt | Stop-Quantity reduzieren, ggf. Stop auf Break-Even |
| Stop getriggert | Alle offenen Targets stornieren |
| Partial Fill auf Stop | Restmenge + alle Targets sofort Market schliessen |

> **Details:** ARCH-07 (OMS)

---

## 7. Risk Management

### 7.1 Drei Verteidigungslinien

**Linie 1 — Per-Trade:**

| Regel | Wert |
|-------|------|
| Stop-Loss-Distance | **2.2 x ATR(14)** auf 5-Min-Bars (Stakeholder-Entscheidung: Faktor 2.2 ersetzt bisherigen 1.5). ATR wird bei Entry eingefroren und fuer die gesamte Trade-Dauer verwendet |
| Max. Positionsgroesse | 50% des Tageskapitals |
| Min. Risk/Reward-Ratio | 1 : 1.5 |
| Max. Slippage-Toleranz | 0.3% vom Entry-Preis |

> **ATR-Freeze und Faktor 2.2 (Stakeholder-Entscheidung):** Der ATR-Wert wird zum Zeitpunkt des Entries eingefroren (`entryAtr`). Der Faktor 2.2 (statt vormals 1.5) wird fuer drei Berechnungen verwendet: (1) Initiale Stop-Distance = `2.2 x entryAtr`, (2) R-Normalisierung: 1R = `2.2 x entryAtr`, (3) Basis fuer die Profit-Protection-Trail-Berechnung. Der breitere Faktor verhindert vorzeitiges Ausstoppen bei normalem Intraday-Noise.

**Linie 2 — Tages-Ebene (global ueber alle Pipelines UND alle Zyklen):**

| Regel | Wert | Scope |
|-------|------|-------|
| Hard-Stop | -10% Tageskapital (realisiert + unrealisiert). Echtzeit-Pruefung | Global |
| Max. Round-Trip-Trades/Tag | 5 | **Global** ueber alle Pipelines und Zyklen |
| Max. Zyklen pro Pipeline/Tag | 3 (konfigurierbar) | Per Pipeline |
| Cooling-Off | 30 Minuten nach 3 Verlusten in Folge | Global |

> **Limit-Hierarchie:** Das globale 5-Round-Trip-Limit dominiert ueber das per-Pipeline-Cycle-Limit. Beispiel: Bei 3 Pipelines mit je 2 Zyklen waeren 6 Round-Trips theoretisch moeglich — das globale Limit von 5 greift vorher. In der Praxis ist das globale Limit das schaerfere Constraint.

**Linie 3 — System-Ebene:**

| Regel | Wert |
|-------|------|
| Broker-seitige GTC-Stops | Redundanz bei System-Ausfall |
| Heartbeat-Monitor | 3 x 30s = 90s → Kill-Switch |
| Daten-Feed-Monitor | Kein Tick seit > 60s → Kill-Switch |

### 7.2 Position Sizing: Fixed Fractional Risk

| Schritt | Formel |
|---------|--------|
| 1. Risikobetrag | `max_risk = min(Tageskapital x 3%, remaining_budget x 0.8)` |
| 1b. FX-Conversion | `max_risk_usd = max_risk_eur x EUR/USD_spot` |
| 2. Stop-Distance | `stop_dist = 2.2 x entryAtr` (ATR eingefroren bei Entry) |
| 2b. Stress-Adjusted | `effective_stop_dist = stop_dist x stress_factor` (1.0 normal, 1.3 bei HIGH_VOL) |
| 3. Stueckzahl | `shares = floor(max_risk_usd / effective_stop_dist)` |
| 4. Cap | Max. 50% Tageskapital |
| 5. Confidence-Skalierung | `shares = shares x min(regime_confidence, quant_score)` |

### 7.3 Dynamisches Tages-Budget

```
remaining_budget = (Tageskapital x 10%) - abs(realisierte_verluste) - abs(unrealisierte_verluste)
```

Gewinne erhoehen das Budget NICHT. Die Safety Margin (0.8) stellt sicher, dass ein einzelner Trade den Hard-Stop nicht durchbrechen kann.

### 7.4 Multi-Cycle Risk-Budget

| Aspekt | Single-Trade-Day | Multi-Cycle-Day |
|--------|-------------------|-----------------|
| Positionsgroesse | Standard-Sizing | Zyklus 2+: Reduziert (Default 60%) |
| Risk-Budget pro Zyklus | Gesamtes Tagesbudget | Restbudget nach realisiertem P&L |
| Max. Zyklen | 1 | 3 (konfigurierbar) |

### 7.5 Kill-Switch

Der Kill-Switch ist mehrfach implementiert:
1. **Software** — Agent → DAY_STOPPED, alle Positionen per Market-Order schliessen
2. **Broker-seitig** — IB Account-Limits als Fallback
3. **Hardware** — Externer Watchdog
4. **Manuell** — Tastendruck, REST-Endpoint, CLI-Command

**Kill-Switch-Ownership:** odin-data eskaliert, odin-core entscheidet.

### 7.6 Crash-Recovery

ODIN ist in v1 NICHT intraday-restartable fuer Trading. Bei Crash:
- GTC-Stops schuetzen offene Positionen beim Broker
- WinSW startet Prozess automatisch neu
- Startup-Recovery erkennt Crash → **Safe-Mode** (kein Trading, nur Dashboard)
- Naechster regulaerer Tagesstart: Flat-Check Reconciliation

### 7.7 Security und Secrets

| Bereich | Regelung |
|---------|---------|
| Broker-Credentials | Ausschliesslich via Environment-Variablen. Nie in Code, Config-Files oder Logs |
| LLM API-Keys | Rotation alle 30 Tage. Separater Key pro Environment (Dev/Paper/Live) |
| LLM-Kontext | Keine Account-Groesse, persoenliche Daten oder historische P&L im Prompt |
| Netzwerk | Outbound-Whitelist: nur IB TWS Gateway + LLM API. Kein allgemeiner Internet-Zugang |
| Audit-Logs | Append-Only mit Hash-Chain. Taeglich signierte Kopie auf separaten Storage |
| Trade-Intent-Signierung | HMAC-Signatur auf jedem Trade-Intent (Instrument, Richtung, Stueckzahl, Preis, Timestamp) |

### 7.8 Backtesting und Simulation

**Validierungspipeline (Release-Gate):**

| Stufe | Modus | Dauer | Kriterium |
|-------|-------|-------|-----------|
| 1. Paper Trading | Simulated Orders gegen echte Marktdaten (IB-Paper-Account) | Min. 20 Handelstage | Simulated P&L positiv, Drawdown < 10%, Win-Rate > 40% |
| 2. Klein-Live | Echtes Trading mit 10% des Normal-Kapitals (1.000 EUR) | Min. 20 Handelstage | P&L positiv, keine unerwarteten Failure Modes |

**Kostenmodell (MUSS in allen Simulationen verwendet werden):**

| Kostenkomponente | Modellierung |
|-----------------|-------------|
| Kommissionen | IB-Tiered: 0.0035 USD/Share (min. 0.35 USD, max. 1% Handelswert) |
| Spread-Kosten | Halber durchschnittlicher Bid-Ask-Spread |
| Slippage | 1 Tick ueber Spread fuer Entries, 0.5 Tick fuer Exits. Bei Volumen < 50.000/Tag: 2 Ticks |
| FX-Kosten | 0.002% (IB-typischer FX-Spread) |

**Fill-Modell (BrokerSimulator):** Der BrokerSimulator MUSS ein realistisches Fill-Modell verwenden. Fills erfolgen auf dem naechsten verfuegbaren Bar-Preis nach Order-Submission, NICHT auf dem Signal-Bar. Partial Fills werden in Simulation nicht modelliert (erst in Klein-Live mit echten Fills).

**Walk-Forward Validation:** Trainingsperiode 60 Handelstage, Validierungsperiode 20 Handelstage, Rolling Window alle 20 Tage. Parameter MUESSEN in mindestens 3 von 4 aufeinanderfolgenden Validierungsperioden profitabel sein.

**LLM-Response-Caching:** Alle LLM-Responses werden gecacht (Cache-Key: Hash aus System-Prompt-Version + Kontext + Tick-Payload). Im Backtesting MUSS der CachedAnalyst verwendet werden fuer deterministische Reproduzierbarkeit.

> **Details:** ARCH-00 (Systemuebersicht), ARCH-10 (Deployment)

---

## 8. KPI Engine

### 8.1 Indikator-Toolkit

| Indikator | Timeframe | Verwendung |
|-----------|-----------|-----------|
| EMA(9) | 1-Min | Kurzfristiger Trend |
| EMA(21) | 1-Min | Trend-Kreuzung |
| RSI(14) | 5-Min | Ueberkauft/Ueberverkauft |
| ATR(14) | 5-Min | Volatilitaet, Stop-Distance, Sizing |
| Bollinger Bands(20,2) | 5-Min | Ueberhitzung, Runner-Trailing |
| ADX(14) | 5-Min | Trendstaerke |

**Derived Values:**
- `ATR-Decay-Ratio = current_5min_ATR / morning_5min_ATR`
- `Volumen-Ratio = lastBarVolume / SMA(barVolume, 20)`

**ATR-Decay-Trigger:** Wenn `ATR-Decay-Ratio < 0.40` (d.h. aktuelle ATR ist auf weniger als 40% der Morning-ATR gefallen, was einem Decay von ueber 60% entspricht), MUSS die Strategie von Trend-Following auf konservativeres Verhalten umschalten: groessere Decision-Intervalle, kleinere Positionen, Mean-Reversion bevorzugen. Dieser Trigger wird dem LLM als Kontext mitgegeben.

### 8.2 VWAP

VWAP ist **Source-of-Truth ausschliesslich in odin-data**. Die KPI-Engine konsumiert ihn aus dem MarketSnapshot.

### 8.3 Data Quality Gates

Alle Marktdaten durchlaufen vor Einspeisung in den Rolling Buffer Quality Gates in fester Reihenfolge:

1. **Bar Completeness** — OHLCV vollstaendig, Volume > 0. Unvollstaendige Bars verwerfen
2. **Time Sync** — System-Clock vs. Exchange-Timestamp Drift > 2s → Warnung
3. **L2 Integrity** — Bid > Ask (invertiertes Book)? Spread > 10%? → L2-Daten verwerfen
4. **Stale Quote Detection** — Kein neuer Tick seit > 30s waehrend RTH → Warnung. > 60s → Kill-Switch
5. **Crash Detection** — Preis > 5% in < 1 Min UND Volume > 3x Durchschnitt → Bar als Crash-Signal markieren, an Rules Engine weiterleiten (NICHT verwerfen)
6. **Outlier Filter** — Preis > 5% in < 1 Bar UND KEIN Crash-Signal UND normales Volume → Bar verwerfen

> **Reihenfolge MUSS eingehalten werden:** Crash-Detection laeuft VOR dem Outlier-Filter, damit extreme aber reale Marktbewegungen nicht faelschlich gefiltert werden.

**Benoetigte IB-Subscriptions:** reqRealTimeBars (5s-Bars), reqHistoricalData (1-Min/5-Min), reqMktDepth (L2), reqMktData (Ticks), reqOpenOrders, reqAccountUpdates, reqPnL/reqPnLSingle.

### 8.4 Quant-Validierung: Scoring-Modell

| Check | Gewichtung |
|-------|-----------|
| Trend-Alignment (EMA, VWAP) | 0.25 |
| Overbought/Sold (RSI) | 0.20 |
| Volumen-Bestaetigung | 0.20 |
| Spread-Pruefung | 0.15 |
| Risk/Reward-Ratio | 0.20 |

**Schwelle:** `quant_score >= 0.50` → erlaubt. **Veto ist absolut:** Einzelne Hard-Veto-Checks (z.B. Spread > 0.5%) blockieren unabhaengig vom Gesamtscore.

### 8.5 Warmup

Indikatoren gelten erst als valid nach `period + 1` Bars. `warmupComplete = false` → Rules Engine erzeugt keinen Intent. Decision-Cycle laeuft trotzdem (gleicher Codepfad, Guard in Rules Engine).

### 8.6 ADX als Entry-Filter (geplant)

ADX > 20-25 als Trend-Qualitaetsfilter fuer TREND_UP-Entries. Backtest/Plateau-Test erforderlich vor Aktivierung.

> **Details:** ARCH-04 (KPI-Engine)

---

## 9. Datenmodell

### 9.1 Entity-Modell (9 Entities)

| Entity | Modul | Beschreibung |
|--------|-------|-------------|
| TradingRunEntity | odin-execution | Ein Run pro Instrument/Tag |
| TradeEntity | odin-execution | Entry-bis-Exit Round-Trip |
| FillEntity | odin-execution | Broker-Ausfuehrung |
| ConfigSnapshotEntity | odin-execution | Reproduzierbarkeit |
| DecisionLogEntity | odin-brain | Decision-Cycle-Protokoll |
| LlmCallEntity | odin-brain | LLM-Protokollierung + Cache |
| EventRecordEntity | odin-audit | Flaches Append-Only-Archiv |
| PipelineStateEntity | odin-core | State-Recovery |
| DailyPerformanceEntity | odin-core | Tages-Aggregat |

### 9.2 Zwei Persistierungspfade

1. **Direkte Domaenen-Persistierung** — Domaenen-Module schreiben strukturierte Daten synchron ueber eigene Repositories (DDD)
2. **AuditEventDrainer** — Konsumiert EventLog-Spool, schreibt NUR EventRecords (flaches Archiv)

### 9.3 Events

Events sind **immutable**. Alle entscheidungsrelevanten Events (MarketData, Broker-Events, LLM-Calls, Decisions, State-Transitions) werden persistiert. `runId` ist der universelle Join-Key.

**Multi-Cycle-Erweiterungen:**
- `cycleNumber` als Feld in relevanten Events
- Cycle-Summary als aggregierter Event am Ende jedes Zyklus
- Tages-Summary aggregiert ueber alle Zyklen

### 9.4 DDD-Modulschnitt

Entities leben in ihren fachlichen Domaenen-Modulen. `odin-persistence` = reine Infrastruktur (DataSource, JPA, Flyway). Kein zentrales Persistence-Modul fuer fachliche Inhalte.

> **Details:** ARCH-08 (Datenmodell)

---

## 10. Frontend

### 10.1 Kommunikation

| Protokoll | Zweck | Richtung |
|-----------|-------|----------|
| **SSE** | Monitoring-Streams (Pipeline-State, Kurse, P&L, Alerts) | Server → Client |
| **REST POST** | Controls (Kill-Switch, Pause/Resume) | Client → Server |

**Kein WebSocket.** SSE fuer unidirektionales Streaming, REST fuer seltene Control-Aktionen.

### 10.2 SSE-Streams

| Endpoint | Daten |
|----------|-------|
| `/api/stream/instruments/{instrumentId}` | Pipeline-State, Kurs, Indikatoren, P&L, Orders, LLM-Status |
| `/api/stream/global` | Globaler Risk-Status, aggregierte P&L, Kill-Switch-State, Alerts |

### 10.3 REST-Endpoints

| Endpoint | Zweck |
|----------|-------|
| `POST /api/controls/kill` | Kill-Switch Request |
| `POST /api/controls/pause/{id}` | Pipeline pausieren |
| `POST /api/controls/resume/{id}` | Pipeline fortsetzen |
| `GET /api/runs/{runId}` | TradingRun-Details |
| `GET /api/runs/{runId}/decisions` | DecisionLog |
| `GET /api/runs/{runId}/llm-history` | LLM-Call-Historie |

### 10.4 Komponenten

- **Chart:** TradingView Lightweight Charts (Candlestick, Volumen, Indikatoren, Entry/Exit-Marker)
- **Pipeline-Status-Panel:** State, Position, Orders, P&L pro Instrument
- **Global Dashboard:** AccountRiskState, aggregierte P&L, Kill-Switch-State
- **Alert-Panel:** Eskalationsstufen (INFO → WARNING → CRITICAL → EMERGENCY)
- **Controls:** Kill-Switch-Button, Pause/Resume pro Pipeline

### 10.5 Alert-Routing und Eskalation

| Alert-Level | Beispiele | Routing |
|-------------|----------|---------|
| INFO | Tages-Report, Trade-Summary | Dashboard + Email (EOD) |
| WARNING | Hohe Slippage, LLM-Retry, DQ-Gate-Verletzung | Push-Notification |
| CRITICAL | Kill-Switch ausgeloest, LLM-Ausfall, Daten-Feed-Ausfall | Push + SMS/Call |
| EMERGENCY | "Cannot Liquidate", Account-Restriction, System-Crash | SMS + Call + Auto-Escalation |

### 10.6 SLOs (Service Level Objectives)

| Metrik | SLO |
|--------|-----|
| System-Uptime (waehrend RTH) | 99.5% pro Monat |
| LLM-Response-Latenz (p95) | < 5 Sekunden |
| Order-Latenz (Intent → Broker) | < 500ms |
| Kill-Switch-Latenz | < 2 Sekunden |

### 10.7 Tech-Stack

- React 18+, TypeScript strict
- Feature-basierte Ordnerstruktur (DDD)
- CSS Modules, Dark Theme, Desktop-only
- Union-Types statt enum-Keyword, Discriminated Unions fuer SSE-Events
- Feature→Feature-Imports verboten

> **Details:** ARCH-09 (Frontend), Frontend-Guardrails

---

## 11. Offene Punkte / Roadmap

### 11.1 Implementierungs-Roadmap

| Phase | Scope | Status |
|-------|-------|--------|
| **P0** | R-basierte Profit-Protection, Highwater-Mark Trail, ATR-Freeze 2.2, Trail ab 1.5R, Regime-Trail-Weite | Teilweise implementiert |
| **P1** | LLM-Schema erweitern (exit_bias, trail_mode, profit_protection_profile, target_policy), TacticalPolicy-Resolver, TACTICAL_EXIT | Offen |
| **P2** | exhaustion_signal, Subregime-Resolver, Anti-LLM-Drift Monitoring | Offen |
| **P3** | entry_timing_bias, scale_out_profile | Offen |
| **P4a** | FLAT_INTRADAY State, Cycle-Counter, EventLog-Erweiterung | Offen |
| **P4b** | ALLOW_RE_ENTRY, Recovery-Trade Entry-Guards | Offen |
| **P4c** | Risk-Budget zyklenuebergreifend, Reduced Sizing | Offen |
| **P4d** | Scalp/Quick-Trade (Zyklus-Typ 3) | Spaeter |

### 11.2 Nicht im Scope (v1)

- Short-Selling
- Options/Futures
- Multi-Broker-Support
- Cloud-Deployment
- Mobile App
- Vision-Modul (Chart-Bild-Analyse) — nach erfolgreicher Validierung des zahlenbasierten Kerns

### 11.3 Offene Architekturentscheidungen

| Frage | Status |
|-------|--------|
| `target_policy = TRAIL_ONLY` vs. bestehende Tranchierung | Klaerungsbedarf: Empfehlung nur fuer Runner-Anteil oder nach OMS-Anpassung |
| Runner-Trail Struktur (EMA/HL) vs. einheitliches R/MFE-System | Entschieden: Beides als Stop-Kandidaten in `max()` beibehalten |
| ADX als Entry-Filter | Backtest/Plateau-Test erforderlich vor Aktivierung |

### 11.4 Sensitivitaetsanalyse

Fuer die kritischen Parameter `stopLossAtrFactor`, `trailingStopAtrFactor` und `vwapOffsetAtrFactor` SOLL ein systematischer Plateau-Test durchgefuehrt werden:
- Jeden Parameter um +/- 10% variieren
- Plateau = robust (Performance-Kurve flach)
- Nadel = Overfit (Performance bricht bei minimaler Aenderung ein)

---

## Anhang: Vorrangregel bei Dokumentenkonflikten

1. **Dieses Gesamtkonzept** ist die normative Quelle der Wahrheit fuer alle strategischen und fachlichen Entscheidungen
2. **Stakeholder-Entscheidungen** (in diesem Dokument markiert) haben Vorrang vor aelteren Dokumenten
3. **Architekturkapitel 0-10** sind autoritativ fuer technische Implementierungsdetails
4. **Fachkonzept v1.5** gilt operativ weiter fuer Bereiche, die hier als "Geplant" markiert sind, bis die Umsetzung erfolgt
5. **Strategie-Optimierung-Sparring** ist die detaillierte Referenz fuer Profit-Protection, Tactical Parameter Controller und Multi-Cycle-Day

## Anhang: Glossar

| Begriff | Bedeutung |
|---------|----------|
| ATR | Average True Range — Mass fuer Volatilitaet |
| Decision-Bar | 1-Minuten-Bar als primaerer Decision-Trigger |
| EOD-Flat | End-of-Day Flat — alle Positionen am Tagesende geschlossen |
| FSM | Finite State Machine — Zustandsmaschine |
| GTC | Good-Til-Cancel — Order bleibt aktiv bis explizit storniert |
| Hard-Stop | Absolutes Verlustlimit (-10% Tageskapital) |
| Highwater-Mark | Stop kann nur steigen, nie fallen |
| Kill-Switch | Notfall-Mechanismus: Sofort alle Positionen schliessen |
| MFE | Maximum Favorable Excursion — hoechster unrealisierter Gewinn |
| OCA | One-Cancels-All — Broker-seitige Order-Gruppe (NICHT verwendet fuer Stop/Target) |
| Quant-Only-Modus | Degraded Mode ohne LLM: Keine neuen Trades |
| R | Risk-Multiple: 1R = Distanz Entry zu Initial Stop |
| RTH | Regular Trading Hours (US: 09:30–16:00 ET) |
| Runner | Kleiner Restbestand (10-20%) mit Trailing-Stop fuer Trend-Days |
| Scaling-Out | Stufenweiser Positionsabbau in R-basierten Tranchen |
| TACTICAL_EXIT | LLM-gesteuerter Exit (bounded, KPI-bestaetigt) |
| VWAP | Volume Weighted Average Price — Source-of-Truth in odin-data |
