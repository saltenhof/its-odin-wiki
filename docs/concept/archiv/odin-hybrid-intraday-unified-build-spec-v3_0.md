# ODIN/IA — Hybrid Intraday Trading System (Equities, Long-only)
## Unified Concept & Build Spec (v3.0)

**Stand:** 2026-02-20  
**Ziel:** Ein vollständiges, implementierbares Fach-/Algorithmuskonzept für ein hybrides Intraday-Aktiensystem (QuantEngine + LLM auf Augenhöhe), basierend auf **1-Minuten-OHLCV** als einzigem Marktdaten-Input (plus Volume).

---


## 0. Normative Sprache, Status, Leserführung

### 0.1 Normative Sprache

Dieses Dokument nutzt RFC-2119-artige Modalverben:

- **MUSS / DARF NICHT / NIE**: zwingend, Abweichung = Defekt
- **SOLL**: starke Empfehlung, Abweichung nur mit dokumentierter Begründung
- **KANN**: optional

### 0.2 Status-Tags

- **(Normativ)**: Build-Requirement, muss implementiert sein
- **(Geplant)**: vorgesehen, aber nicht kritisch für das nächste Release
- **(Entscheidung)**: stakeholderseitig festgelegt (nicht “wegzuoptimieren” ohne neue Entscheidung)

### 0.3 Was „vollständig“ hier bedeutet

„Vollständig“ meint: Das Dokument deckt **alle** Entwicklungsrelevanten Punkte ab:

- Scope/Constraints
- Signal-Hierarchie (1m/3m/10m) und Entscheidungslogik
- Rollen/Verantwortlichkeiten (QuantEngine, LLM, Arbiter, Risk/OMS)
- Datenarchitektur + Data-Quality-Gates
- OMS/Execution-Policy ohne Orderbuchdaten
- Backtesting/Validierung inkl. Reproduzierbarkeit
- Model Risk Management (Prompt-/Schema-Versionierung, Regression)
- Security/Operational Safety (Kill-Switch, Audit)
- Monitoring/Logging + UI/Controls (anbieteragnostisch)

---


## 1. Executive Summary

### 1.1 Was ist das System?

Ein vollautomatischer Intraday-Trading-Agent für **Aktien**, **Long-only**, der pro Instrument/Tag typischerweise einen Haupt-Trade managt, dabei aber:

- **Entry** nach Opening-Konsolidierung finden kann,
- **Teil-Exits** bei parabolischen Anstiegen ausführen kann,
- nach Pullback/Bottom **Re-Entry** bzw. **Aufstocken** in einem Multi-Cycle-Tag unterstützt,
- und am Ende des Tages **EOD-flat** ist.

### 1.2 Kerndesign: Hybrid auf Augenhöhe (Entscheidung)

- **QuantEngine**: deterministisch, schnell, nachvollziehbar; wertet KPIs und Pattern-Features aus.
- **LLM**: liefert Kontext-/Regimebeurteilung und Parameter-Bias, erkennt „ungewöhnliche“ Muster; aber strikt **bounded**.
- **Arbiter**: fusioniert deterministisch (Dual-Key-Mechanismen, Vetos, Hysterese).
- **Risk/OMS**: ist eine eigenständige Verteidigungslinie; kann beide „überstimmen“, wenn Limits verletzt sind.

> Ergebnis: Adaptivität ohne Kontrollverlust.

### 1.3 Systemparameter & Rahmenbedingungen (Normativ, konfigurierbar)

| Parameter | Default | Kommentar |
|---|---:|---|
| Tageskapital | 10.000 EUR | Aufgeteilt auf aktive Pipelines |
| Max. Tagesverlust (Hard-Stop) | -10% | global über alle Pipelines |
| Instrumente parallel | 1–3 | je Pipeline genau 1 Instrument |
| Richtung | Long-only | kein Short |
| Instrumenttyp | Aktien | keine Derivate |
| Datenbasis | 1-Min OHLCV + Volume | kein Orderbuch, keine Breadth |
| Rohsignal | 1m | Monitoring-Ebene |
| Quant-Decision-Frame | 3m | resampled |
| Confirmation-Frame | 10m | Trend-/Regime-Hysterese |
| Handelszeiten | Exchange-konfigurierbar | typ. US RTH + optional Pre |
| Autonomie | Vollautomatisch | mit manuellen Controls (Pause/Kill) |
| EOD-Flat | MUSS | keine Overnight-Position |

**Instrumentauswahl:**  
- Instrumente werden **extern vorgegeben** (Scanner/Watchlist außerhalb dieses Systems).
- Auswahl wird zu Session-Start „eingefroren“ (kein Symbolwechsel intraday), damit Backtests/Logs reproduzierbar bleiben.

---


## 2. Scope, Nicht-Ziele, harte Constraints

### 2.1 In Scope (Normativ)

- Intraday Long-Trades auf Aktien
- Ein- und Ausstieg, Teilverkäufe, Trailing-Stops, Re-Entry/Scale-In
- Multi-Cycle-Handling (mehrere Episoden pro Tag pro Instrument), aber begrenzt
- Degradation Modes (z. B. LLM-Ausfall)
- Backtesting + Walk-Forward + Paper/Klein-Live Pipeline

### 2.2 Nicht-Ziele / Out-Of-Scope

- Short, Optionen, Futures
- Portfolio-Optimierung / Korrelationen über viele Instrumente
- HFT / Sekunden-Scalping
- Orderbuch-/Spread-Arbitrage
- News-getriebene Strategie (News höchstens optional als Metadaten)

### 2.3 Harte Constraints (Entscheidung)

- Marktdaten: **nur 1-Minuten-OHLCV inklusive Volumen** (plus ableitbare Daily-Kontexte).
- Keine bid/ask, kein L2, keine Marktbreite -> Architektonisch vorzusehen, für spätere Ausbaustufen geplant.
- System muss dennoch robust auf **Extremsituationen** reagieren (Sprünge nach oben/unten, Shock Events, Blow-off-Tops, Flushes).

---


## 3. Zeitauflösung, Datenpipeline, Resampling

### 3.1 Zeit & Trading-Kalender (Normativ)

- Alle Zeiten beziehen sich auf die Börsen-Zeitzone des Instruments (z. B. `America/New_York` für US).
- **MarketClock** ist die einzige Zeitquelle im Trading-Codepfad.  
  Kein direkter Zugriff auf Systemzeit in Decision-/Risk-/OMS-Logik.
- Trading-Kalender MUSS Feiertage, halbe Handelstage, DST-Umstellungen korrekt abbilden.

### 3.2 Timeframes & Rollen (Normativ)

| Frame | Quelle | Rolle |
|---|---|---|
| **1m** | direkt | Monitoring, Extrembewegungen, Event-Detektoren, stop tightening |
| **3m** | resampled | Quant-Decision Frame: Entry/Exit/Scale-Entscheidungen |
| **10m** | resampled | Trend-/Regime-Confirmation, Anti-Flip/Hysterese |
| **Daily** | aus Historie | ADR(14), Gap, Prior Day Levels |

**Entscheidung:** Rohsignal 1m; QuantEngine rechnet primär auf 3m (klarere Signale); 10m als Quercheck.

### 3.3 Resampling-Regeln (Normativ)

Resampling erfolgt **zeitbasiert und bar-close-basiert**:

- Eine 3m-Bar ist gültig erst nach Close der dritten 1m-Bar.
- Aggregation:
  - Open = Open der ersten 1m-Bar
  - High = max(High)
  - Low = min(Low)
  - Close = Close der letzten 1m-Bar
  - Volume = Sum(Volume)

**Alignment:**  
- Resampling ist an Session-Start ausgerichtet (z. B. 09:30:00 → 09:32:59, 09:33:00 → 09:35:59).

### 3.4 Rolling Buffers (Normativ)

Minimale Buffergrößen je Pipeline:

- 1m: 240 Bars (≈ 4h)
- 3m: 200 Bars (≈ 10h)
- 10m: 80 Bars (≈ 13h)
- Daily: 30 Tage

### 3.5 VWAP aus OHLCV (Normativ)

Da kein Tick-VWAP verfügbar ist, wird bar-basiert approximiert:

- `typical_price = (High + Low + Close) / 3`
- `VWAP_day = sum(typical_price * Volume) / sum(Volume)`

VWAP ist Bestandteil jedes MarketSnapshot und wird im EventLog versioniert.

### 3.6 Data Quality Gates (Normativ)

**Wichtig:** Reihenfolge ist entscheidend. Crash-/Extrem-Detection MUSS vor Outlier-Filter laufen, damit echte Extrembewegungen nicht „weggefiltert“ werden.

Empfohlene Reihenfolge (nur OHLCV, ohne L2):

1. **Bar Completeness**  
   - OHLC vorhanden, Volume vorhanden (darf 0 sein bei manchen Märkten; konfigurieren)  
   - sonst: Bar verwerfen, `DQ_BAR_INCOMPLETE`
2. **Time Sync / Monotonicity**  
   - barTimestamp strikt monoton  
   - Drift MarketClock vs BarTimestamp > threshold → WARNING
3. **Stale Bar Detection**  
   - keine neue 1m Bar innerhalb X Sekunden nach erwarteter Zeit → WARNING  
   - länger anhaltend → Trading Halt (Degradation)
4. **Crash/Spike Detection (markieren, nicht verwerfen)**  
   - Preisänderung > P% in 1–2 Bars UND VolumeRatio > V  
   - → `EVENT_CRASH_OR_SPIKE` erzeugen, Bar bleibt im Buffer
5. **Outlier Filter (verwerfen)**  
   - Preisänderung > P% in 1 Bar UND VolumeRatio „normal“  
   - → vermutlich Datenfehler; Bar verwerfen, `DQ_OUTLIER_REJECTED`
6. **Session Boundary Guard**  
   - außerhalb Session: Bars je nach Regel (Pre/Post) akzeptieren oder ignorieren

> Parameter P/V sind konfigurierbar und werden in der Challenger-Suite kalibriert.

---


## 4. Architektur: Module, Ports, Verantwortlichkeiten

### 4.1 Vier-Schichten-Architektur (Normativ)

1. **Datenschicht**
   - Marktdaten empfangen, DQ-Gates, Rolling Buffers, Snapshots
2. **Entscheidungsschicht (Brain)**
   - QuantEngine (KPIs + Pattern), LLM Analyst, Strategy-FSM, Arbiter
3. **Risiko- & Ausführungsschicht**
   - Risk Gate, OMS (Orders/Stops/Targets), ExecutionGateway
4. **Präsentation & Monitoring**
   - Event-Streams, Dashboards, Controls (Pause/Kill), Reporting

### 4.2 Fünf zentrale Ports (Normativ)

Diese Ports sind die Mindest-Schnittstellen, damit Live und Simulation denselben Codepfad nutzen können:

| Port | Verantwortung | Live-Adapter | Sim-Adapter |
|---|---|---|---|
| **MarketClock** | Zeit, Kalender, Session-Phasen | SystemClock (Exchange TZ) | SimClock (Replay) |
| **MarketDataFeed** | liefert 1m Bars | LiveFeedAdapter | HistoricalReplayFeed |
| **ExecutionGateway** | Orders/Fills/Cancels | BrokerAdapter | BrokerSimulator |
| **LlmAnalyst** | liefert JSON-Analyse | LLMClientAdapter | CachedAnalyst/Stub |
| **EventLog** | append-only persistieren | Persistenter Store | Gleicher Store + Barrier |

**Entscheidung:** Simulation ist First-Class-Concern: Unterschiede nur über Adapter.

### 4.3 Domain-Module & Ownership (Normativ)

- **data-domain**: RollingBuffer, MarketSnapshot, DQ events
- **brain-domain**: Strategy FSM, Arbiter, QuantEngine, LLM-Protokoll
- **execution-domain**: Risk Gate, OMS, Execution policy
- **observability-domain**: EventLog schema, reports, dashboards

**Persistenzprinzip:**  
- Domain-Module dürfen strukturierte Entities persistieren (Repositories).  
- Zusätzlich existiert ein flaches **AuditEvent-Archiv** (EventRecords).  
- `runId` ist universeller Join-Key.

### 4.4 Verantwortlichkeits-Duo (Quant ⇄ LLM) (Entscheidung)

- Quant und LLM liefern beide **Votes** und **Constraints**, aber:
  - LLM darf **nie** direkte Order-Details (Stückzahl, Stop-Level, Limit-Preis) bestimmen.
  - Quant darf **nie** „blind“ handeln ohne Regime-/Kontext-Konsistenzchecks (Hysterese).

Der **Arbiter** ist deterministisch und hat die endgültige Entscheidung über **Intent** (ENTER/EXIT/…),
während **Risk/OMS** die endgültige Entscheidung über **Ausführung** und **Safety** hat.

---


## 5. Datenmodell, Events, Audit-Trail

### 5.1 Warum Events „first-class“ sind (Normativ)

Das System MUSS jeden entscheidungsrelevanten Schritt in Events festhalten, um:

- Replays deterministisch zu ermöglichen (Backtest/Forensik),
- Entscheidungen nachvollziehbar zu machen (Warum Entry?),
- Drift und Fehlverhalten zu erkennen (LLM, Datenfeed, Execution).

### 5.2 Identitäten & Join Keys (Normativ)

- `runId`: Instrument × Handelstag × Pipeline
- `cycleNumber`: 1..N (Multi-Cycle-Episoden)
- `decisionId`: pro Decision-Bar (3m) eindeutig
- `llmCallId`: pro LLM-Aufruf eindeutig
- `orderId` / `fillId`: OMS/Broker IDs

### 5.3 Minimales Entity-Modell (Normativ)

| Entity | Muss-Felder (Auszug) |
|---|---|
| TradingRun | runId, instrumentId, exchange, session, configHash, start/end |
| Cycle | runId, cycleNumber, start/end, realizedPnl |
| Position | runId, qty, avgPrice, unrealizedPnl, highwater |
| Order | orderId, runId, cycleNumber, side, qty, type, limit, status |
| Fill | fillId, orderId, price, qty, timestamp |
| DecisionLog | decisionId, featureSnapshots, votes, arbiterDecision, reasons |
| LlmCall | llmCallId, promptVersion, inputHash, outputJson, validationResult, latency |
| Alert | severity, code, message, timestamp |

### 5.4 Event-Taxonomie (Normativ)

**Market/Data**
- `Bar1mReceived`
- `Bar3mClosed`
- `Bar10mClosed`
- `DataQualityEvent` (code)

**Features**
- `FeatureSnapshot1m`
- `FeatureSnapshot3m`
- `FeatureSnapshot10m`

**Hybrid Votes**
- `QuantVote` (regime + intent + conf + vetos)
- `LlmVote` (regime + bias + conf + redFlags)

**Decision/State**
- `DecisionCycleStarted`
- `IntentProposed`
- `IntentApproved`
- `IntentRejected` (reason codes)
- `StateTransition`

**OMS/Execution**
- `OrderSubmitted`
- `OrderCanceled`
- `OrderRepriced`
- `OrderFilled` / `OrderPartiallyFilled`
- `StopPlaced` / `StopUpdated`
- `TargetPlaced` / `TargetFilled`

**Risk/Safety**
- `RiskGateRejected`
- `DailyHardStopTriggered`
- `KillSwitchTriggered`
- `DegradationModeChanged`

**EOD**
- `CycleSummary`
- `RunSummaryEod`

### 5.5 Reason Codes (Normativ)

Jede Blockade muss maschinenlesbar begründet werden, z. B.:

- `DQ_BAR_INCOMPLETE`, `DQ_OUTLIER_REJECTED`, `DQ_STALE_FEED`
- `WARMUP_INCOMPLETE`, `SESSION_ENTRY_BLOCKED`, `EOD_ENTRY_BLOCKED`
- `RISK_DAILY_LIMIT`, `RISK_RR_TOO_LOW`, `RISK_MAX_TRADES`
- `HYBRID_DUAL_KEY_FAIL`, `QUANT_HARD_VETO`, `LLM_VETO_ENTRY`
- `LLM_SCHEMA_INVALID`, `LLM_TIMEOUT`, `LLM_TTL_EXPIRED`

### 5.6 Tamper-Proof Audit (Normativ, empfohlen aus IA/MC)

- EventLog ist append-only.
- Jeder Eintrag enthält Hash des vorherigen Eintrags (**Hash-Chain**).
- Zusätzlich tägliche signierte Kopie auf separaten Storage.

### 5.7 Trade-Intent Signierung (Geplant, aber sinnvoll)

- Jeder finaler Intent (ENTER/EXIT/…) wird als Payload signiert (HMAC o. ä.).
- Execution-Schicht validiert Signatur vor Order-Submission.
- Ziel: Nachweisbarkeit gegen Manipulation zwischen Brain und Execution.

---


## 6. KPI Engine & Quantitative Validierung

### 6.1 Indikator-Toolkit (Normativ)

Alle Indikatoren werden aus OHLCV berechnet.

| Indikator | Frame | Zweck |
|---|---|---|
| EMA(9) | 3m | kurzfristiger Trend |
| EMA(21) | 3m | Trend-Kreuzung / Pullback-Qualität |
| RSI(14) | 3m | Overbought/Oversold-Veto, Momentum |
| ATR(14) | 3m | Stop-Distance, Sizing, Vol-Regime |
| ADX(14) | 3m | Trendqualität (Filter) |
| VWAP (bar-basiert) | intraday | Mean-/Trend-Anchor |
| Volume Ratio | 1m/3m | Bestätigung/Climax-Detektion |
| Range/ATR Ratio | 1m/3m | Extrembewegungen/“zu wild” |

**Confirmation (10m)** nutzt weniger Indikatoren, dient primär der Hysterese.

### 6.2 Derived Values (Normativ)

- `ATR_Decay_Ratio = current_ATR_10m / morning_ATR_10m`  
  (oder 3m→10m konsistent; wichtig ist: ein „Morning Baseline“-Fenster)
- `Volume_Ratio = lastVolume / SMA(volume, 20)`
- `DistToVWAP_ATR = (close - vwap) / ATR`

### 6.3 Volatilitätsabnahme im Tagesverlauf (Normativ)

**Trigger:** Wenn `ATR_Decay_Ratio < 0.40` (Beispielschwelle, zu kalibrieren), MUSS das System umschalten:

- Decision-Frequenz wird konservativer (z. B. keine „Immediate“ Decisions ohne CRITICAL Event)
- Positionsgröße wird reduziert
- Mean-Reversion-/Reclaim-Setups priorisiert, aggressive Trend-Chasing deaktiviert

Dieser Trigger geht als Kontext in LLM-Calls und als Flag in QuantEngine ein.

### 6.4 Pattern-Feature-Erkennung (Normativ)

Pattern-Features werden deterministisch aus Rolling Buffers abgeleitet und als Events/Flags dem Brain geliefert:

| Feature | Berechnung (Beispiel) | Verwendung |
|---|---|---|
| Flush-Detection | Range > 2×ATR UND Close nahe Low UND Volume_Ratio > X | Setup „Flush→Reclaim“ |
| Reclaim-Detection | Nach Flush: Close > VWAP UND Close > MA-Cluster binnen N Bars | Transition FLUSH→RECLAIM |
| Coil-Detection | fallende Hochs + steigende Tiefs + ATR sinkt | Breakout-Setup |
| Higher-Low | lokales Low > vorheriges Low (Swing-Analyse) | Trendbestätigung/Re-Entry |
| Breakout-Strength | Breakout-Range / avgRange(phase) | Fakeout-Filter |
| Pullback-Depth | Pullback in ATR-Einheiten | „gesund“ vs strukturbrechend |

> Swing-Analyse MUSS ohne Lookahead implementiert werden (nur bestätigte Swings).

### 6.5 Quant-Validierung: Scoring + Hard-Vetos (Normativ)

QuantEngine liefert sowohl einen **Score** als auch **Hard-Vetos**.

**Score-Komponenten (Beispiel, anpassbar):**
- Trend alignment (EMA + VWAP): 0.25
- Momentum/Overbought (RSI): 0.20
- Volume confirmation: 0.20
- Liquidity proxy check (Volume/Range): 0.15
- Risk/Reward Check: 0.20

**Schwelle:** `quant_score >= 0.50` → grundsätzlich erlaubt.  
**Hard-Veto:** einzelne Checks können absolut blockieren (z. B. „zu illiquide“, „EOD Entry blocked“).

### 6.6 Warmup (Normativ)

Indikatoren sind erst gültig nach `period + 1` Bars (je Frame).  
Vor `warmupComplete == true` darf die Rules Engine **keinen** Entry-Intent erzeugen (Decision Loop läuft trotzdem).

---


## 7. Tages-Lifecycle, Pipeline-FSM, Decision Loop

### 7.1 Tagesphasen (Normativ, konfigurierbar)

Beispiel für US RTH (in Exchange TZ):

| Phase | Zeitfenster | Verhalten |
|---|-------------|---|
| Pre-Market (optional) | 07:00–09:30 | Beobachten, ggf. strengere Regeln |
| Opening Auction/Spike | 09:30–09:40 | **kein Entry**, Baseline-Kalibrierung |
| Active Trading | 09:40–15:15 | Decision Loop aktiv, Setups erlaubt |
| Power Hour | 15:15–15:30 | strengere Gates, neue Entries nur noch selektiv |
| Entry Cutoff | ab 15:30    | **keine neuen Entries** |
| Forced Close | 15:45–15:55 | Positionen schließen (Policy) |
| End-of-Day | 15:55–16:15 | Reconciliation, Reports, Calibration |

### 7.2 Pipeline-Zustandsmaschine (FSM) (Normativ)

| State | Zweck | Trading erlaubt? |
|---|---|---|
| INITIALIZING | Start, Config-Snapshot, Buffer init | Nein |
| WARMUP | Indikatoren füllen | Nein (Entry block) |
| OBSERVING | Markt lesen, keine Position | Ja (nur wenn Gates) |
| SEEKING_ENTRY | aktives Entry-Scannen | Ja |
| PENDING_FILL | Order offen | Nein (nur mgmt/cancel) |
| MANAGING_TRADE | Position offen | Ja (Scale/Exit) |
| COOLDOWN | nach Exit, Overtrade-Schutz | Nein |
| DAY_STOPPED | Hard-Stop / Kill / severe DQ | Nein |
| FORCED_CLOSE | EOD liquidation | Nein (nur closing) |
| EOD | Abschluss | Nein |

**Multi-Cycle:** `cycleNumber` wird bei `COOLDOWN → SEEKING_ENTRY` erhöht.

### 7.3 Decision Loop (3m) — deterministische Hauptschleife (Normativ)

Bei jedem Close einer 3m-Bar:

1. DQ Gates check
2. FeatureSnapshot3m berechnen
3. QuantVote erzeugen (Regime + Intent + Score/Vetos)
4. LLMVote anfordern oder Cache/TTL prüfen
5. Arbiter fusioniert (Dual-Key + Hysterese)
6. RiskGate prüft Budget/Sizing/Stop
7. OMS/Execution setzt Orders & Schutz
8. Alle Schritte als Events loggen

**Pseudocode (fachlich):**
```text
on Bar3mClosed:
  if state in {DAY_STOPPED, FORCED_CLOSE, EOD}: return
  dq = runDQ()
  if dq.failCritical: enter(DAY_STOPPED)

  features = computeFeatures3m()
  quant = quantEngine.vote(state, features, monitorEvents, confirmation10m)

  llm = llmManager.getVoteOrRequest(state, features, context, ttlPolicy)

  arbiterDecision = arbiter.decide(state, quant, llm, riskBudget, sessionPhase)

  if arbiterDecision.intent != NO_ACTION:
     if riskGate.allows(arbiterDecision):
        oms.execute(arbiterDecision)
     else:
        log IntentRejected(RISK_...)
```

### 7.4 1m Monitoring Loop (Normativ)

Bei jeder 1m-Bar:

- aktualisiert 1m Features (schnell)
- detektiert Events:
  - Crash/Spike
  - Parabolic acceleration
  - Exhaustion/Climax
  - VWAP reclaim
- kann triggern:
  - **Immediate Re-Evaluation** (außerhalb 3m-Takt) bei CRITICAL
  - Stop/Trail tightening (OMS maintenance)
  - LLM refresh (wenn TTL abläuft & Ereignis stark)

**Regel:** 1m-Monitor darf nicht final handeln, sondern nur:
- Stop-/Risk-Protect auslösen,
- oder eine Decision Cycle sofort anfordern.

### 7.5 10m Confirmation Loop (Normativ)

Bei jedem Close einer 10m-Bar:

- bestätigt Trend-/Regime (Hysterese)
- aktualisiert RegimePersistenz
- setzt „DayType“-Flags (TrendDay, RangeDay, HighVolDay)
- wirkt als Gate im Arbiter

---


## 8. Hybrid-Protokoll: QuantEngine ⇄ LLM ⇄ Arbiter

### 8.1 Grundprinzip (Entscheidung)

- QuantEngine und LLM liefern **beide** eine Lagebeurteilung.
- Der Arbiter fusioniert deterministisch.
- Risk/OMS hat Safety-Priorität.

### 8.2 LLM ist bounded (Normativ)

LLM darf:
- Regime-/Kontextlabels liefern,
- taktische Bias (ALLOW/CAUTION/VETO) liefern,
- bounded Parameter-Modifikationen vorschlagen (innerhalb Clamps),
- RedFlags melden.

LLM darf NICHT:
- Stückzahlen, Orderpreise, Stop-Level, Ordertypen bestimmen,
- unstrukturierte Texte als Entscheidungsoutput liefern (nur Schema).

### 8.3 Vote-Schemata (Normativ)

#### 8.3.1 QuantVote

MUSS enthalten:
- `regime_label` (UP/DOWN/RANGE/HIGH_VOL/UNCERTAIN)
- `regime_confidence` (0..1)
- `intent_vote` (ENTER/ADD/SCALE_OUT/EXIT/NO_ACTION)
- `intent_confidence` (0..1)
- `quant_score` (0..1) + Score-Komponenten
- `hard_vetos` (Liste Enums)
- Baseline-Parameter (trail_width_atr, stop_atr_mult, etc.)

#### 8.3.2 LlmVote

MUSS enthalten:
- `regime_label` (UP/DOWN/RANGE/HIGH_VOL/UNCERTAIN)
- `regime_confidence` (0..1)
- `intent_bias` (ALLOW_ENTRY/NEUTRAL/CAUTION/VETO_ENTRY/EXIT_BIAS/SCALE_OUT_BIAS)
- `urgency` (LOW/MEDIUM/HIGH/CRITICAL)
- `param_modifiers` (bounded deltas)
- `red_flags` (bounded enums)
- `ttl_seconds` (vorgeschlagen, Arbiter clamp’t)

### 8.4 Dual-Key Regeln (Normativ)

**Entry** ist Dual-Key:
- QuantEngine muss Entry erlauben (Score >= threshold, keine Hard-Vetos)
- LLM darf nicht vetoen

**Exit**:
- Hard Stops / Forced Close: rein deterministisch (LLM nicht erforderlich)
- Tactical Exit/Scale-out: mindestens ein starker Trigger + keine Quant-Hard-Vetos gegen Exit (Exit soll nicht blockiert werden)

**Re-Entry/Scale-In**:
- erfordert Confirmation (10m) und reduziert sizing (Risk Policy)

### 8.5 Konfliktauflösung (Normativ)

Wenn Quant und LLM widersprechen:

1. **Safety zuerst**: Risk/OMS kann immer blocken.
2. **Entry-Konflikt**:
   - Quant ENTER, LLM VETO → kein Entry
   - Quant NO_ACTION, LLM ALLOW_ENTRY → kein Entry (LLM kann nicht allein entry erzwingen)
3. **Exit-Konflikt**:
   - Quant EXIT, LLM NEUTRAL/CAUTION → Exit ist erlaubt, wenn Risk/Policy ok
   - Quant hält, LLM EXIT_BIAS + CRITICAL → Arbiter kann „Scale-Out“ priorisieren statt Voll-Exit (Konservativ)

### 8.6 Parameter-Fusion (Normativ)

- Quant liefert Baseline.
- LLM liefert bounded delta.
- Arbiter clamp’t final.

Beispiel:
- Baseline trail_width = 1.0 ATR
- LLM delta = +0.2 ATR (HIGH_VOL)
- Clamp: [0.6, 1.6] → final = 1.2 ATR

### 8.7 LLM Call-Management (Normativ)

- **Single-flight** pro Pipeline: niemals zwei parallele LLM-Calls für dieselbe Entscheidung.
- **Caching**: jede LLM-Response wird gecacht (Key = promptVersionHash + inputHash).
- **TTL/Freshness**:
  - Entry braucht frische LLM-Analyse (z. B. < 120s)
  - Management darf länger (z. B. < 900s)
- **Timeouts**:
  - Timeout → Degradation (Quant-only, keine neuen Entries)

### 8.8 Anti-Halluzination & Input Hardening (Normativ)

- Schema-Validation: invalid → reject, `LLM_SCHEMA_INVALID`
- Plausibilität: Confidence darf nicht immer konstant hoch sein → Drift-Alert
- Konsistenz: LLM darf nicht UP labeln, wenn Confirmation10m eindeutig DOWN ist (dann auto-override auf CAUTION)
- Prompt Injection: Wenn optionale externe Inputs genutzt werden, müssen sie als untrusted markiert werden (kein direkter Steuerbefehl)

---


## 9. Strategielogik: Regime, Setups, Multi-Cycle

### 9.1 Regime-Definitionen (Normativ)

Regime werden als Labels mit Confidence geführt:

- **UP**: Preis > VWAP, EMA9>EMA21 (3m), Higher-High/Higher-Low, ADX↑
- **DOWN**: Preis < VWAP, EMA9<EMA21, Lower-Lows
- **RANGE**: VWAP-Magnet, ADX niedrig, häufige Richtungswechsel
- **HIGH_VOL**: große Ranges, schnelle Swings, Spike/Crash-Events
- **UNCERTAIN**: widersprüchlich / zu wenig Daten / DQ warnings

### 9.2 Regime-Hysterese (Normativ)

Regimewechsel wird nur akzeptiert wenn:

- 10m Confirmation: 2 Bars konsistent **oder**
- 1× 10m + 1× CRITICAL Event (z. B. Crash+Reclaim)

Ziel: Nicht bei jedem VWAP-Cross flippen.

### 9.3 Setup-Katalog (MVP) (Normativ)

MVP umfasst mindestens diese 4 Setups:

- **Setup A**: Opening Konsolidierung → Trend Entry
- **Setup B**: Flush → Reclaim → Run (V-Reversal)
- **Setup C**: Coil/Compression → Breakout
- **Setup D**: Re-Entry / Aufstocken nach Korrektur

Jedes Setup wird als **State Machine** implementiert (deterministisch), mit:

- Conditions für EntryGate
- Management-Regeln (Stops/Targets/Scale)
- Invalidation & Exit-Regeln
- erlaubte Re-Entry-Bedingungen

### 9.4 Setup A — Opening Konsolidierung → Trend Entry

**Intent:** nicht im ersten Chaos kaufen, sondern nach Beruhigung.

**States:**
- `A_OBSERVE_OPEN`
- `A_WAIT_CONSOLIDATION`
- `A_READY`
- `A_IN_TRADE`
- `A_DONE`

**Detektion (1m, Opening-Window):**
- `OPENING_SPIKE` wenn:
  - 1m Range/ATRproxy hoch
  - VolumeRatio hoch
- `CONSOLIDATION` wenn:
  - Range-Kompression über N Bars
  - Preis hält VWAP oder reclaim’t VWAP
  - keine neuen Lows

**EntryGate (3m, nach Opening-Block):**
- quant_score >= threshold
- regime != DOWN confirmed (10m)
- LLM intent_bias != VETO_ENTRY
- Entry-Preis in „reasonable zone“ (nicht > X ATR über VWAP)

**Trade Management:**
- initial stop: unter Struktur (Swing Low) oder ATR-multiple
- scale-out bei 1R/2R oder bei Parabolic/Exhaustion Event
- runner mit trailing stop

### 9.5 Setup B — Flush → Reclaim → Run

**Intent:** starke Abwärtsübertreibung + schnelle Erholung nutzen, aber Fake-Bounces vermeiden.

**States:**
- `B_NEUTRAL`
- `B_FLUSH_DETECTED`
- `B_RECLAIM`
- `B_CONFIRM_RUN`
- `B_IN_TRADE`
- `B_DONE`

**FLUSH_DETECTED (1m):**
- großer Down-Move in kurzer Zeit + Volume spike
- Close nahe Low (seller climax)

**RECLAIM (3m):**
- Close reclaim’t VWAP oder MA-Cluster
- keine neuen Lows seit Flush

**CONFIRM_RUN (10m):**
- Higher-Low bestätigt + Trendfilter dreht

**Entry:**
- Starter-Entry in RECLAIM (kleiner)
- Add erst in CONFIRM_RUN (wenn bestätigt)

**Invalidation:**
- neues Low unter Flush-Low → sofortiger Exit/Abort

### 9.6 Setup C — Coil/Compression → Breakout

**Intent:** aus enger Konsolidierung einen explosiven Breakout handeln.

**States:**
- `C_NEUTRAL`
- `C_COIL_FORMING`
- `C_BREAKOUT_TRIGGERED`
- `C_RETEST_OPTIONAL`
- `C_IN_TRADE`
- `C_DONE`

**COIL_FORMING (3m/10m):**
- fallende Hochs + steigende Tiefs
- ATR sinkt
- Volumen moderat oder fallend

**BREAKOUT (3m):**
- Close außerhalb Range + VolumeRatio > X
- LLM red_flag fakeout_risk? → entscheidet Retest vs Sofortentry (als Bias)

**Management:**
- schnelle Profit Protection, da Coil oft zu Blow-offs führt
- scale-out Ladder bei Parabolic

### 9.7 Setup D — Re-Entry / Aufstocken nach Korrektur

**Intent:** nach vorherigem Exit oder nach frühem Downtrend späteren Trendwechsel handeln.

**States:**
- `D_ELIGIBLE` (nur wenn Cycle>0 oder Vormittag ohne Trade)
- `D_BOTTOMING`
- `D_RECLAIM`
- `D_CONFIRM`
- `D_IN_TRADE`
- `D_DONE`

**Gates:**
- Cooldown abgelaufen
- 10m nicht DOWN confirmed
- Reclaim VWAP + Higher-Low
- LLM erkennt Regimewechsel (bias shift)

**Sizing:** reduziert (z. B. 60–80% von Cycle1), aggressivere Profit Protection.

### 9.8 Multi-Cycle-Day Regeln (Normativ)

- Max Zyklen pro Pipeline/Tag: Default 3
- Cycle2+ sizing reduziert
- Globales Round-Trip-Limit dominiert (z. B. 5 pro Tag global)

### 9.9 Ground-Truth Labeling für Regime-Kalibrierung (Normativ)

Für Kalibrierung von Regime-Confidences wird ex-post ein Ground-Truth erzeugt (EOD), ohne Leakage:

- Window: 30 Minuten ab Zeitpunkt t
- Kriterien (Beispiel):
  - TREND_UP: Close(t+30m) > Close(t) + 0.5×ATR und höhere Hochs/Tiefs
  - TREND_DOWN: Close(t+30m) < Close(t) - 0.5×ATR und tiefere Hochs/Tiefs
  - RANGE: |Close(t+30m)-Close(t)| < 0.3×ATR
  - HIGH_VOL: Range(window) > 2×ATR
  - UNCERTAIN: sonst

Damit werden Konfidenzen (Quant & LLM) über Zeit kalibriert (Calibration Curve).

---


## 10. Risk Management, Sizing, Guardrails

### 10.1 Drei Verteidigungslinien (Normativ)

**Linie 1: Trade-Ebene**
- max risk pro Trade (fixed fractional)
- min R/R
- stop distance bounds (zu eng/zu weit blocken)

**Linie 2: Tages-Ebene**
- globaler Hard-Stop (z. B. -10% Tageskapital)
- max Round-Trips/Tag global
- max Zyklen/Tag/Pipeline
- Cooling-Off nach Verlustserie

**Linie 3: System-Ebene**
- DataFeed Heartbeat
- LLM availability
- Broker errors
- Kill-Switch & Safe Mode

### 10.2 Dynamisches Tagesbudget (Normativ)

Budget basiert nur auf Verlusten; Gewinne erhöhen das Budget nicht.

Beispiel:
```text
remaining_budget = (daily_capital * daily_loss_limit) 
                   - abs(realized_losses) 
                   - abs(unrealized_losses)
```

Zusätzlich Safety-Margin:
- `effective_budget = remaining_budget * 0.8`

Ziel: ein einzelner Trade kann Hard-Stop nicht durchbrechen.

### 10.3 Position Sizing: Fixed Fractional Risk (Normativ)

**Schritte (beispielhaft, anpassbar):**
1. Risikobetrag:
   - `max_risk = min(daily_capital * 0.03, effective_budget)`
2. Stop-Distance:
   - `stop_dist = stop_atr_mult * entry_ATR` (ATR bei Entry eingefroren)
   - `effective_stop_dist = stop_dist * stress_factor`  
     stress_factor = 1.0 normal, 1.3 bei HIGH_VOL
3. Stückzahl:
   - `shares = floor(max_risk / effective_stop_dist)`
4. Cap:
   - max Positionswert (z. B. 50% Tageskapital)
5. Confidence-Skalierung:
   - `shares *= min(regime_confidence, quant_score)` (clamp’t)

**Cycle2+**: sizing multipliziert mit 0.6–0.8 (konfigurierbar).

### 10.4 Stop-Policy (Normativ)

- Initial Stop MUSS sofort nach Fill gesetzt werden.
- Stops dürfen **nie gelockert** werden (nur tighten).
- Trailing basiert auf:
  - Highwater-Mark (Preis) und
  - optional Struktur (Higher-Low) und
  - ATR-basierte Trail-Width.

### 10.5 Profit Protection (Normativ)

Ab definierten MFE/R-Schwellen:
- ab 1R: Stop mindestens Break-even
- ab 2R: Teilgewinn realisieren oder trail enger (Policy)
- bei Parabolic/Exhaustion: aggressive scale-out + trail tighten

### 10.6 Kill-Switch (Normativ)

Kill-Switch ist mehrfach implementiert:

1. **Software**: Pipeline → DAY_STOPPED, Orders cancel, Position schließen (Policy)
2. **Broker-seitig**: broker-Stop/limits als Fallback (wenn verfügbar)
3. **Manuell**: Control-Interface (Button/Endpoint/CLI)

Kill-Switch erzeugt EMERGENCY Alert + Audit Event.

### 10.7 Crash-Recovery (Normativ)

Wenn Prozess abstürzt:

- Broker-seitige Stops schützen Position (wenn möglich).
- Nach Restart: System startet in **Safe Mode** (kein Trading), nur Dashboard.
- Nächster regulärer Tagesstart führt Flat-Reconciliation durch.

### 10.8 Degradation Modes (Normativ)

| Situation | Mode | Verhalten |
|---|---|---|
| LLM Timeout / invalid schema | QUANT_ONLY | keine neuen Entries, bestehende Position managen, EOD close |
| Data feed stale | DATA_HALT | keine neuen Orders, ggf. risk-reducing exit |
| Broker rejects/cannot liquidate | EMERGENCY | Kill-Switch, manuelle Eskalation |

---


## 11. OMS & Execution Policy (ohne Orderbuchdaten)

### 11.1 Grundprinzip (Normativ)

- OMS ist deterministisch, reproduzierbar.
- Keine Mikrosekunden-Optimierung; Orders werden auf 1m/3m Takt gesteuert.
- LLM liefert maximal **Dringlichkeit** und **Bias**, aber keine Orderdetails.

### 11.2 Order-Arten (Normativ)

- **Entry Orders**: Limit (Standard), optional Stop-Limit (Breakout)
- **Exit Orders**:
  - Stop (schutz)
  - Targets (Tranche)
  - Forced Close (Market oder aggressive Limit) in Close-Out-Phase

### 11.3 Bracket-Logik (Normativ)

Nach erfolgreichem Entry-Fill MUSS OMS atomar:

1. Schutzstop platzieren
2. Targets/Tranchen platzieren (falls Policy)
3. Trailing-Policy aktivieren

> „Atomar“ heißt: im Eventlog als zusammenhängender Schritt; bei Fehlern Recovery-Mechanismus.

### 11.4 Multi-Tranche Modell (Normativ)

Position wird in Tranchen geführt:

- Tranche1 (early): Gewinn früh sichern (z. B. 20–30%)
- Tranche2/3: Profit Taking bei Momentum
- Runner: kleiner Rest (z. B. 10–20%) mit Trailing Stop

Tranche-Auslösung:
- R-Multiples (1R, 2R, 3R)
- oder Parabolic/Exhaustion Events

### 11.5 Repricing Policy ohne Echtzeit-Quotes (Normativ)

Da nur 1m Bars verfügbar sind:

- Repricing maximal einmal pro neuer 1m Bar (oder pro 3m Decision Cycle).
- Repricing ist bounded: max N Reprice-Versuche, max price drift in ATR.
- Wenn nach maxReprice kein Fill: Order cancel, Intent zurück an Arbiter (state bleibt SEEKING_ENTRY).

### 11.6 Fill Handling (Normativ)

OMS MUSS:

- Partial Fills korrekt behandeln (Position/Stops/Targets anpassen)
- Idempotent sein (doppelte Fill-Events dürfen keine doppelte Position erzeugen)
- Broker-Rejections als CRITICAL Alert behandeln

### 11.7 Stop Update Cadence (Normativ)

Stops werden aktualisiert:
- auf jedem 1m Close (wenn trailing aktiv)
- oder bei CRITICAL Monitor Event (z. B. Blow-off → tighten)

### 11.8 Forced Close Policy (Normativ)

- Ab `forcedCloseStart` werden keine neuen Entries angenommen.
- Offene Positionen werden sukzessive geschlossen:
  - bevorzugt aggressive Limit
  - fallback Market (wenn riskant, nicht liquidierbar)
- „Cannot liquidate“ → EMERGENCY Alert + Kill-Switch + Eskalation

---


## 12. LLM-Integration: Prompt-Architektur, Schema, Safety

### 12.1 Rollenmodell: LLM als Analyst (Normativ)

LLM ist **Analyst** und **Regime-Sensor**, nicht Trader.

- Es liefert Lagebeurteilung + Bias + RedFlags.
- Es erzeugt keinen finalen Trade-Intent und keine Orderdetails.

### 12.2 Input-Design (Normativ)

LLM erhält ausschließlich:

- komprimierte Markt-/Feature-Historie (1m/3m/10m)
- Session-Phase + State
- Position/Trade-Kontext (ohne sensitive Daten)
- Risk-Budget als kategoriale Klasse (LOW/MED/HIGH), nicht exakter Kontostand

**Datensparsamkeit (Security):**
- keine Account-Größe, keine persönlichen Daten, kein historisches P&L im Prompt
- keine Secrets (API keys etc.)

### 12.3 Prompt-Architektur (Normativ)

- **System Prompt**: Rolle, Regeln, Output-Schema, Verbote (keine Orderdetails)
- **Context Prompt**: Tageskontext, Regime-/Volatilitätsflags, ggf. ATR-decay
- **Market Snapshot**: letzte Bars + Features in kompakter Form
- **Memory**: letzte Decisions + Outcomes (begrenzt)

### 12.4 Output-Schema (Normativ)

LLM MUSS JSON liefern, z. B.:

- `regime_label` (enum)
- `regime_confidence` (0..1)
- `intent_bias` (enum)
- `urgency` (enum)
- `param_modifiers` (bounded deltas)
- `red_flags` (enum list)
- `explanation_short` (max 1–2 Sätze, optional, rein fürs Dashboard)

**Schema Validation** ist hart: invalid = reject.

### 12.5 Caching & Deterministischer Replay (Normativ)

- Jede LLM-Response wird gecacht.
- Cache-Key = Hash(SystemPromptVersion + SchemaVersion + InputPayloadHash).
- Backtests MÜSSEN den CachedAnalyst nutzen, um deterministischen Replay zu gewährleisten.

### 12.6 TTL & Freshness (Normativ)

- Entry-Entscheidungen dürfen nur auf LLM Outputs basieren, deren TTL nicht abgelaufen ist.
- TTL wird abhängig von Vol-Regime gesetzt (HIGH_VOL → kürzer).

### 12.7 Drift Monitoring (Normativ)

- Verteilung von `regime_confidence` über Zeit überwachen (wenn immer 0.9 → Alert)
- Agreement Rate Quant vs LLM überwachen
- Impact-Analyse: wie oft LLM-Veto Trades verhindert, wie oft es Exits beschleunigt

---


## 13. Backtesting & Validierungspipeline (Release-Gate)

### 13.1 Ziele (Normativ)

- Realistisches Ausführungsmodell auf OHLCV
- Keine Lookahead-Leakage
- Reproduzierbarkeit (Replay) inkl. LLM
- Robustheit gegen Overfitting (Walk-Forward, Sensitivität)

### 13.2 Grundregel: Kein „Signal-Bar-Fill“ (Normativ)

Orders dürfen im Backtest nicht auf derselben Bar gefüllt werden, auf der das Signal entsteht,
sondern frühestens auf der nächsten verfügbaren Bar/Preis (Policy definiert).

### 13.3 Fill-Modell (OHLCV) (Normativ)

**Limit Order Fill (konservativ):**
- Wenn Limit innerhalb [Low, High] der **nächsten** Bar liegt → fill
- Fill-Preis:
  - konservativ: ungünstiger Preis innerhalb Bar
  - oder: Limit-Preis (weniger konservativ)
- Wenn nicht erreicht → kein Fill, repricing/cancel gemäß OMS-Policy

**Stop Fill:**
- Wenn Stop durch Gap übersprungen → fill zum Open der nächsten Bar (worst-case)

### 13.4 Kostenmodell ohne Spread (Normativ)

Da kein bid/ask:

- Kommission: konfigurierbar (pro share / pro Order / prozentual)
- Slippage Proxy:
  - `slippage = base_slippage + k * ATR * volatility_factor`
  - volatility_factor abhängig von Range/ATR und VolumeRatio
- Zusätzlich „Shock Slippage“ bei Crash/Spike Events

> Wichtig: gleiches Kostenmodell in Backtest, Paper, Klein-Live (Vergleichbarkeit).

### 13.5 Walk-Forward Validation (Normativ)

- Rolling windows (Beispiel):
  - Train: 60 Handelstage
  - Validate: 20 Handelstage
  - Schrittweite: 20 Tage
- Parameter gelten als robust, wenn sie in ≥ 3 von 4 Validierungsfenstern profitabel sind
  und Risk-Metriken innerhalb Limits liegen.

### 13.6 Ablation Tests (Normativ)

MUSS mindestens:

- Quant-only
- LLM-only (nur als Experiment, nicht produktiv)
- Hybrid

auswerten, um den Mehrwert des LLM objektiv zu messen.

### 13.7 Release-Gate Pipeline (Normativ)

Empfohlenes Gate:

1. **Backtest Suite** (historisch)  
2. **Paper Trading** (simulated orders gegen echte Marktdaten) — min 20 Handelstage  
3. **Klein-Live** (z. B. 10% Kapital) — min 20 Handelstage  
4. **Live** (voll)

Jede Stufe hat Abbruchkriterien (Drawdown, Incident-Rate, Unexpected Failure Modes).

### 13.8 Report-Artefakte (Normativ)

Jeder Run erzeugt:
- EOD Summary (P&L, Trades, MAE/MFE, Slippage Proxy)
- Regime Calibration Update (Ground Truth vs Pred)
- LLM/Quant Agreement Stats
- Incident Log (DQ, LLM timeouts, broker rejects)

---


## 14. Model Risk Management (LLM) & Change Management

### 14.1 Versionierung (Normativ)

- `prompt_version` (semver)
- `schema_version` (semver)
- `strategy_version` (semver)
- `cost_model_version` (semver)

Alle Versionen werden pro Run im ConfigSnapshot geloggt.

### 14.2 Regression-Gates (Normativ)

Vor Deployment einer neuen Version MUSS:

- Challenger-Suite (Kap. 16) durchlaufen
- Backtest über repräsentative Perioden laufen
- LLM Schema-Compliance 100% (sonst reject)
- Drift Checks: Confidence-Verteilungen, Agreement-Rate, etc.

### 14.3 Safety-Contract Tests (Normativ)

Automatisierte Tests stellen sicher, dass LLM:

- keine Orderdetails ausgibt
- keine verbotenen Felder ausgibt
- bei unklaren Situationen UNCERTAIN/CAUTION wählt (nicht overconfident)

### 14.4 „Worst Case: LLM komplett weg“ (Normativ)

System schaltet auf Quant-only:

- Keine neuen Trades (Entry blockiert)
- Bestehende Positionen werden deterministisch gemanagt
- EOD wird geschlossen

### 14.5 Monitoring in Produktion (Normativ)

- LLM Timeout Rate
- Schema Invalid Rate
- Response Latency p95
- Anteil LLM-Vetos
- Performance-Delta Hybrid vs Quant-only (Shadow Mode möglich)

---


## 15. Security, Isolation, Operational Safety

### 15.1 Secrets (Normativ)

- Broker-Credentials nur über Environment/Secret-Store, nie in Code/Logs.
- LLM API Keys separat pro Environment; Rotation regelmäßig.
- Logs dürfen keine Secrets enthalten (Log-Scrubber).

### 15.2 Netzwerk-Isolation (Geplant, empfohlen)

- Outbound-Whitelist: nur DataFeed/Broker/LLM-Endpunkte.
- Kein allgemeiner Internetzugang im Trading-Prozess.

### 15.3 Audit-Logs (Normativ)

- Append-only + Hash-Chain (siehe Kap. 5.6)
- Retention-Policy: definierter Zeitraum
- tägliche signierte Kopie

### 15.4 Heartbeats & Watchdogs (Normativ)

- DataFeed Heartbeat: wenn Bars ausbleiben → WARNING → HALT
- Process watchdog: wenn Prozess abstürzt → restart in Safe Mode
- OMS/Broker heartbeat: fehlende Bestätigungen → ALERT

### 15.5 Incident-Eskalation (Normativ)

Alert-Level:
- INFO → WARNING → CRITICAL → EMERGENCY

EMERGENCY löst aus:
- Kill-Switch
- manuelle Eskalation (Call/SMS/Push)

---


## 16. Edge Cases & Failure Modes

### 16.1 Markt-Extremfälle (Normativ)

1. **Explosiver Breakout nach langer Konsolidierung**
   - 1m Monitor erkennt Breakout/Acceleration
   - Arbiter erlaubt schnelle Teilgewinne, Trail tighten
2. **News-/Shock-artiger Absturz**
   - Crash/Spike Event
   - Stop/Hard-Risk greift
   - ggf. sofortige Risk-Reducing Exit (ohne LLM)
3. **Gap über Stop**
   - Stop fill zum nächsten Bar-Open (worst-case)
   - Event `GAP_STOP_SLIPPAGE`
4. **„Parabolic + Plateau + Dump into Close“**
   - Exhaustion/Climax Events
   - Scale-out ladder zwingend
   - EOD forced close
5. **Opening flush + schnelle Erholung**
   - Setup B (Flush→Reclaim), Starter Entry nur nach Reclaim

### 16.2 Infrastruktur-Fälle (Normativ)

- Datafeed stottert → DQ_STALE_FEED → DATA_HALT
- LLM antwortet nicht → QUANT_ONLY → keine neuen Entries
- Broker rejects/cannot liquidate → EMERGENCY → Kill-Switch
- Duplicate fills → OMS idempotent

### 16.3 Trading Halts / Session Interrupts (Geplant)

Ohne spezielle Halt-Daten wird ein Halt indirekt erkannt:
- plötzliche Bar-Lücken + keine Updates → HALT Mode
- keine neuen Orders, nur risk-reducing cancels

---


## 17. Monitoring, Logging, Dashboard & Controls

### 17.1 Monitoring-Streams (Normativ)

Das System MUSS in Echtzeit bereitstellen:

- Pipeline State (FSM)
- aktueller Kurs (1m), VWAP, EMA, RSI, ATR, ADX (mind. die wichtigsten)
- Position, unrealized/realized P&L
- offene Orders, Stops, Targets
- LLM Status (letzter Call, TTL, latency, validation)
- Alerts

### 17.2 Control-Interface (Normativ)

Minimale Controls:

- `kill` (global)
- `pause`/`resume` pro Pipeline
- `safe_mode` (read-only, kein Trading)

### 17.3 Empfohlenes Kommunikationsmodell (Geplant)

- **Server → Client**: unidirektionales Streaming (z. B. SSE)
- **Client → Server**: REST/Command-Endpoints

Konkrete Protokolle sind Implementation-Details; wichtig ist die Trennung:
- seltene Control-Kommandos
- häufige Monitoring-Updates

### 17.4 SLOs (Geplant, sinnvoll)

Beispielziele:
- LLM response latency p95 < 5s
- Intent→Broker latency < 500ms (abhängig von Broker)
- Kill-Switch latency < 2s
- Uptime während Session > 99%

### 17.5 Logging-Standards (Normativ)

- Structured Logs (JSON)
- Korrelation über runId, cycleNumber, decisionId
- Keine Secrets
- Log-Level Policy (INFO/WARN/ERROR)

---


## 18. Challenger-Suite: 20 Intraday-Szenarien als Abnahmetests

**Ziel:** Das Konzept ist nur dann „vollständig“, wenn es sich gegen typische und extreme Tagesverläufe behauptet.
Diese Suite ist eine Mischung aus synthetischen Pattern-Tagen und historischen Beispieltagen (wenn Daten vorhanden).

### 18.1 Testformat (Normativ)

Jeder Challenger definiert:

- **Input:** 1m OHLCV Serie (ggf. Daily-Kontext)
- **Erwartete Detektion:** welche MonitorEvents/Flags müssen entstehen?
- **Erwartete erlaubte Aktionen:** Entry/Exit/Scale/Block
- **Erwartete Safety-Reaktionen:** Stop-Updates, Degradation, EOD flat
- **Pass/Fail-Kriterien:** deterministische Bedingungen (nicht nur P&L)

### 18.2 Die 20 Szenarien (MVP) (Normativ)

> Hinweis: Schwellenwerte sind in Parametern definiert; die Suite prüft die Logik, nicht die exakten Werte.

#### S01 — Opening Spike → Konsolidierung → Trend up (Setup A)
- Verlauf: starke erste 5–10 Minuten, dann Range-Kompression, dann Trend.
- Erwartung:
  - Opening-Block verhindert Entry vor Ende Opening Window.
  - CONSOLIDATION erkannt.
  - Entry im 3m Frame nach Freigabe.
  - Profit Protection ab 1R, ggf. Runner.

#### S02 — Opening Spike → Fake Breakout → Range Day
- Verlauf: Spike, kurzer Ausbruch, sofort zurück zur VWAP, Chop.
- Erwartung:
  - LLM oder Quant RedFlag „fakeout/chop“.
  - Entry wird blockiert oder sehr schnell wieder beendet.
  - Overtrade-Schutz (kein ständiges Rein/Raus).

#### S03 — -10% Flush in 5–10 min → V-Reversal → Rally (Setup B)
- Verlauf: schneller Abverkauf, dann Rückeroberung, dann Rally.
- Erwartung:
  - FLUSH_DETECTED Event.
  - Starter-Entry erst nach RECLAIM.
  - Add erst nach Confirmation (10m) / Higher-Low.

#### S04 — -10% Flush → Dead-Cat-Bounce → weiter DOWN
- Erwartung:
  - FLUSH erkannt.
  - Reclaim-Gate wird nicht erfüllt ODER Setup wird abgebrochen.
  - Kein „Averaging down“.

#### S05 — Coil über 2h → Breakout midday (Setup C)
- Erwartung:
  - COIL_FORMING erkannt.
  - Entry entweder Breakout oder Retest abhängig von Fakeout-Risiko.
  - Parabolic-Protection aktiv, wenn Ausbruch explodiert.

#### S06 — Coil → Fakeout → Stop small loss
- Erwartung:
  - Breakout Entry möglich.
  - Hard-Stop greift schnell.
  - Cooldown verhindert sofortigen Re-Entry.

#### S07 — Parabolic Run → Plateau → Dump into Close
- Verlauf: schneller Run, Plateau, dann schneller Abverkauf zum Close.
- Erwartung:
  - Parabolic/Exhaustion Events.
  - Scale-out ladder vor Dump.
  - EOD forced close zuverlässig.

#### S08 — Slow Trend Day (steady up, wenige Pullbacks)
- Erwartung:
  - Kein hektisches Scalping.
  - Runner bleibt, Trailing folgt.
  - Nicht zu früh alles verkaufen.

#### S09 — VWAP Magnet Chop Day
- Erwartung:
  - RANGE Regime.
  - Entry-Gates restriktiv.
  - Max-Trades und Cooldown verhindern Overtrade.

#### S10 — Gap Up & Fade
- Verlauf: Gap up, dann Abverkauf unter VWAP.
- Erwartung:
  - Entry nur nach Reclaim; kein blindes Kaufen in Gap-Extension.
  - Wenn Position offen: schneller Exit, wenn VWAP nicht hält.

#### S11 — Gap Down & Recover (Setup B Variante)
- Erwartung:
  - keine Panik-Entries; nur nach Reclaim+Confirmation.

#### S12 — News-ähnlicher Shock Dump (intraday)
- Erwartung:
  - Crash Event.
  - Stop/Exit ohne LLM.
  - Degradation möglich (wenn Daten unsicher).

#### S13 — Volatility Halt / Datenlücke
- Verlauf: plötzlich fehlen Bars.
- Erwartung:
  - DQ_STALE_FEED, Trading Halt.
  - Keine neuen Orders, bestehende Position nur risk-reducing.

#### S14 — Datafeed liefert Outlier-Bar (Fehler)
- Verlauf: einzelne Bar mit absurd hoher Range, aber ohne Volumen-Anomalie.
- Erwartung:
  - Outlier Filter verwirft Bar.
  - Keine falschen Trades.

#### S15 — LLM outage mitten im Trade
- Erwartung:
  - QUANT_ONLY Mode.
  - Keine neuen Entries.
  - Bestehende Position deterministisch managen, EOD close.

#### S16 — Broker reject bei Order
- Erwartung:
  - OMS erkennt Reject, CRITICAL Alert.
  - Retry nur begrenzt.
  - ggf. Kill-Switch je nach Severity.

#### S17 — Partial fills (simuliert)
- Erwartung:
  - Position/Stops/Targets werden korrekt skaliert.
  - Keine doppelte Exposure.

#### S18 — Re-Entry nach Take Profit + Pullback + neuer Trend (Setup D)
- Erwartung:
  - Cycle2 wird erlaubt nach Cooldown.
  - sizing reduziert.
  - Profit Protection früher.

#### S19 — Kein Entry vormittags (DOWN), Trendwechsel nachmittags
- Erwartung:
  - Setup D wird aktiv.
  - Confirmation (10m) verhindert zu frühes „Catch the falling knife“.

#### S20 — End-of-day squeeze → reversal
- Erwartung:
  - Neue Entries nach cutoff blockiert.
  - Offene Position wird geschützt/geschlossen (EOD flat).

### 18.3 Regression Report (Normativ)

Jeder Challenger erzeugt:
- Pass/Fail je Erwartung
- Timeline der Events/States
- Decisions + ReasonCodes
- (optional) Chart-Overlay aus Eventlog

---


## 19. Appendix A — Parameterkatalog (konfigurierbar)

### 19.1 Session & Zeitparameter

| Parameter | Beschreibung | Default |
|---|---|---|
| opening_no_trade_start | Start Opening-Block | session open |
| opening_no_trade_end | Ende Opening-Block | +15m |
| entry_cutoff_time | keine neuen Entries ab | 15:30 |
| forced_close_start | forced liquidation start | 15:45 |
| forced_close_end | spätestens flat | 15:55 |
| cooldown_after_exit_min | Cooldown nach Exit | 10–30m |
| max_cycles_per_day | max cycle pro pipeline | 3 |
| max_round_trips_global | globales Limit | 5 |

### 19.2 DQ/Extrem-Detection

| Parameter | Beschreibung | Default |
|---|---|---|
| dq_stale_seconds_warn | Warnschwelle, keine Bar | 90s |
| dq_stale_seconds_halt | Halt-Schwelle | 180s |
| crash_move_pct | Move% für Crash/Spike | 5% |
| crash_volume_ratio | VolumeRatio für Crash/Spike | 3.0 |
| outlier_move_pct | Move% für Outlier | 5% |
| outlier_max_volume_ratio | max VolumeRatio für Outlier | 1.2 |

### 19.3 Quant Scoring

| Parameter | Beschreibung | Default |
|---|---|---|
| quant_score_threshold | Gate für Entry | 0.50 |
| weight_trend | EMA/VWAP alignment | 0.25 |
| weight_rsi | overbought/sold | 0.20 |
| weight_volume | volume confirm | 0.20 |
| weight_liquidity_proxy | volume/range filter | 0.15 |
| weight_rr | risk/reward | 0.20 |

### 19.4 Risk/Sizing

| Parameter | Beschreibung | Default |
|---|---|---|
| daily_loss_limit | Hard-Stop | 10% |
| trade_risk_fraction | max risk/trade | 3% |
| budget_safety_margin | safety clamp | 0.8 |
| stop_atr_mult | initial stop distance | 2.2 |
| stress_factor_high_vol | multiplier | 1.3 |
| cycle2_size_factor | sizing cycle2+ | 0.6–0.8 |

### 19.5 LLM

| Parameter | Beschreibung | Default |
|---|---|---|
| llm_timeout_sec | request timeout | 5–10s |
| llm_entry_ttl_sec | TTL für Entry | 120s |
| llm_mgmt_ttl_sec | TTL für Mgmt | 900s |
| llm_debounce_sec | debounce for event-driven calls | 30–60s |

---

## 20. Appendix B — Enumerations & JSON Schema (Skizze)

### 20.1 Enums

- `RegimeLabel`: UP, DOWN, RANGE, HIGH_VOL, UNCERTAIN
- `Intent`: ENTER, ADD, SCALE_OUT, EXIT, NO_ACTION
- `IntentBias`: ALLOW_ENTRY, NEUTRAL, CAUTION, VETO_ENTRY, EXIT_BIAS, SCALE_OUT_BIAS
- `Urgency`: LOW, MEDIUM, HIGH, CRITICAL
- `RedFlag`: CHOP_RISK, FAKEOUT_RISK, EXHAUSTION_RISK, GAP_RISK, DATA_QUALITY_RISK

### 20.2 LLM Output JSON (Beispiel)

```json
{
  "schema_version": "1.0",
  "regime_label": "UP",
  "regime_confidence": 0.72,
  "intent_bias": "ALLOW_ENTRY",
  "urgency": "MEDIUM",
  "ttl_seconds": 120,
  "param_modifiers": {
    "trail_width_atr_delta": -0.1,
    "entry_strictness_delta": 0.2
  },
  "red_flags": ["CHOP_RISK"],
  "explanation_short": "VWAP reclaim + higher lows, vol stabilizing"
}
```

---

## 21. Appendix C — Glossar (Auszug)

- **Pipeline**: isolierter Handelspfad für ein Instrument/Tag
- **Cycle**: Episode innerhalb eines Tages (Entry→Exit)
- **Dual-Key**: Entry braucht Quant-Gate + kein LLM-Veto
- **DQ**: Data Quality
- **Arbiter**: deterministische Fusionslogik
- **EOD-flat**: keine Position über Nacht

---

## 22. Traceability: Abdeckung von IA (Fachkonzept) und MC (Master Concept)

### 22.1 Warum diese Matrix?

Deine Kritik („21 KB vs. 80 KB“) ist valide als Warnsignal: Kürze kann bedeuten, dass Teile fehlen.
Dateigröße ist jedoch kein zuverlässiges Maß für Vollständigkeit — viele Konzepte sind lang, weil sie:
- provider-spezifische APIs enthalten,
- Wiederholungen haben,
- oder sehr breit (Security, UI, Ops, Validierung) ausformulieren.

Diese Matrix macht es explizit: Welche IA/MC-Themen sind hier enthalten?

### 22.2 IA Kapitel → dieses Dokument

| IA Kapitel (v1.5) | Abdeckung in diesem Dokument |
|---|---|
| Systemparameter & Rahmenbedingungen | Kap. 1.3 |
| Designphilosophie | Kap. 0 + 1.2 + 4 |
| Vier-Schichten-Architektur | Kap. 4 |
| Tages-Lifecycle | Kap. 7.1 |
| Zustandsmaschine des Agenten | Kap. 7.2 |
| LLM-Integration & Prompt-Architektur | Kap. 12 |
| Deterministic Strategy Rules | Kap. 9 |
| Quantitative Validierungsschicht | Kap. 6 |
| Entscheidungsschleife (Decision Loop) | Kap. 7.3 |
| Risikomanagement & Guardrails | Kap. 10 |
| Anti-Halluzination & Absicherung | Kap. 8.8 + 12.4 |
| Datenarchitektur & Subscriptions | Kap. 3 + 4.2 + 6.6 |
| Intraday Regime Detection | Kap. 9.1–9.2 + 9.9 |
| Volatilitätsabnahme | Kap. 6.3 |
| Execution Policy | Kap. 11 |
| News Security & Input Validation | Kap. 8.8 + 12.2 (untrusted inputs) |
| Backtesting & Validierungspipeline | Kap. 13 |
| Model Risk Management | Kap. 14 |
| Security & Isolation | Kap. 15 |
| Operationelle Sicherheit | Kap. 10.6–10.8 + 15.4–15.5 |
| Edge Cases & Failure Modes | Kap. 16 |
| Monitoring & Logging | Kap. 17 |
| Appendix | Kap. 19–21 |

### 22.3 MC Kapitel → dieses Dokument

| MC Kapitel (v1.1) | Abdeckung in diesem Dokument |
|---|---|
| Strategielogik | Kap. 9 |
| Multi-Cycle-Day | Kap. 9.8 |
| Trailing Stop & Profit Protection | Kap. 10.4–10.5 |
| LLM-Integration | Kap. 12 |
| OMS | Kap. 11 |
| Risk Management | Kap. 10 |
| KPI Engine | Kap. 6 |
| Datenmodell | Kap. 5 |
| Frontend | Kap. 17.1–17.3 (anbieteragnostisch) |
| Roadmap | Kap. 16–18 + Appendix |

### 22.4 Bewusste Änderungen gegenüber IA/MC (Entscheidung)

- **Keine provider-spezifischen Subscriptions** (wie IB reqRealTimeBars, L2, Ticks).  
  Stattdessen: OHLCV-only DQ Gates + Proxy-Liquidität.
- **Zeitauflösung angepasst:** Rohsignal 1m; Quant-Entscheidung 3m; Confirmation 10m.
- **Hybrid symmetrisch:** IA/MC tendieren teils asymmetrisch (LLM analysiert, Regeln entscheiden).  
  Hier: beide liefern Votes; Arbiter fusioniert deterministisch; Risk/OMS bleibt Safety-Owner.
- **Execution ohne Orderbuch:** Repricing/Stops werden auf 1m/3m Cadence modelliert.

---

## 23. Ergänzungen für Engineering-Umsetzung

Dieses Kapitel enthält bewusst „engineering-nahe“ Spezifikationen, ohne Frameworks zu diktieren.

### 23.1 Arbiter: Entscheidungstabelle (Normativ)

Der Arbiter entscheidet pro Decision Cycle genau **einen** Intent:

- NO_ACTION
- ENTER
- ADD
- SCALE_OUT
- EXIT

**Allowed Actions per State:**

| State | ENTER | ADD | SCALE_OUT | EXIT |
|---|---:|---:|---:|---:|
| WARMUP | ✗ | ✗ | ✗ | ✗ |
| OBSERVING | ✓ | ✗ | ✗ | ✗ |
| SEEKING_ENTRY | ✓ | ✗ | ✗ | ✗ |
| PENDING_FILL | ✗ | ✗ | ✗ | ✓ (Cancel/Abort) |
| MANAGING_TRADE | ✗ | ✓ | ✓ | ✓ |
| COOLDOWN | ✗ | ✗ | ✗ | ✗ |
| DAY_STOPPED | ✗ | ✗ | ✗ | ✓ (force close only) |
| FORCED_CLOSE | ✗ | ✗ | ✗ | ✓ |
| EOD | ✗ | ✗ | ✗ | ✗ |

**Dual-Key Matrix (Entry):**

| QuantVote | LLM Bias | Ergebnis |
|---|---|---|
| ENTER + score ok + keine vetos | ALLOW_ENTRY/NEUTRAL | ENTER möglich |
| ENTER | CAUTION | ENTER nur wenn Confirmation UP + urgency HIGH (optional) |
| ENTER | VETO_ENTRY | BLOCK |
| NO_ACTION | ALLOW_ENTRY | BLOCK |
| EXIT (z. B. OVERBOUGHT) | ALLOW_ENTRY | BLOCK |

> Diese Tabellen verhindern, dass “implizite” Regeln später anders implementiert werden.

### 23.2 OMS: Order State Machine (Normativ)

Orders durchlaufen mindestens diese Zustände:
- NEW → SUBMITTED → (PARTIAL_FILLED)* → FILLED
- NEW/SUBMITTED → CANCELED
- SUBMITTED → REJECTED

**Regeln:**
- Jeder Broker-Event wird idempotent verarbeitet (dedupe by orderId+eventSeq).
- PARTIAL_FILLED aktualisiert Stop/Targets proportional.
- REJECTED ist CRITICAL, wenn Position ungeschützt ist.

### 23.3 Stop/Target Konsistenz (Normativ)

- Es MUSS zu jedem Zeitpunkt gelten:  
  „Wenn Position > 0, dann existiert ein wirksamer Schutzmechanismus“  
  (broker stop oder interner stop + sofortige exit-policy).

### 23.4 Daten-Serialisierung (Normativ)

- MarketSnapshot, FeatureSnapshot, Votes und Decisions sind serialisierbar (z. B. JSON).
- Jede Entscheidung enthält den vollständigen Snapshot-Hash, um später „genau dieselbe Situation“ rekonstruieren zu können.

### 23.5 Konfigurationsstruktur (Beispiel, nicht bindend)

Konfiguration muss versioniert und pro Run gesnapshottet werden.

Beispiel (YAML-ähnlich):
```yaml
session:
  openingBlockMinutes: 15
  entryCutoff: "15:30"
  forcedCloseStart: "15:45"
risk:
  dailyLossLimitPct: 0.10
  tradeRiskPct: 0.03
  stopAtrMult: 2.2
  cycle2SizeFactor: 0.7
dataQuality:
  crashMovePct: 0.05
  crashVolumeRatio: 3.0
llm:
  timeoutSec: 8
  entryTtlSec: 120
  mgmtTtlSec: 900
```

---

## 24. Optional: Minimal API-Spezifikation (anbieteragnostisch)

Wenn ein Dashboard gebaut wird, kann eine API wie folgt aussehen.

### 24.1 Streams (Server→Client)

- `/stream/global`
  - global risk state
  - kill-switch state
  - alerts
- `/stream/instruments/{instrumentId}`
  - pipeline state
  - last bars
  - key indicators
  - orders/position
  - llm status

### 24.2 Controls (Client→Server)

- `POST /controls/kill`
- `POST /controls/pause/{instrumentId}`
- `POST /controls/resume/{instrumentId}`
- `GET /runs/{runId}`
- `GET /runs/{runId}/decisions`
- `GET /runs/{runId}/llm-history`

---

## 25. Checklisten: „Build Readiness“

### 25.1 Data Layer Checklist (MUSS)

- [ ] MarketClock implementiert (live+sim)
- [ ] 1m ingestion + DQ gates + event emission
- [ ] 3m/10m resampling korrekt (alignment, close-only)
- [ ] Rolling buffers + snapshot hashes

### 25.2 Brain Layer Checklist (MUSS)

- [ ] QuantEngine: features + score + hard vetos
- [ ] LLM: schema + cache + TTL + timeout/degradation
- [ ] Arbiter: tables implementiert + reason codes

### 25.3 Execution Layer Checklist (MUSS)

- [ ] OMS state machine + idempotent fills
- [ ] bracket: stop+targets after fill
- [ ] forced close policy
- [ ] kill-switch end-to-end

### 25.4 Validation Checklist (MUSS)

- [ ] backtester uses same decision code path
- [ ] fill + cost model configured
- [ ] challenger suite pass/fail report
- [ ] release gate pipeline documented

---

## 26. Konkrete Event-Detektoren (1m) & Proxy-Signale (ohne Orderbuch)

Dieses Kapitel ist bewusst „algorithmisch konkret“, weil hier oft die größten Missverständnisse entstehen.

### 26.1 Opening Volatility Decay Detector (Normativ)

**Ziel:** Erkennen, wann die erste High-Vola-Phase vorbei ist.

Definitionen:
- `ATR1m_short` = ATR(3) auf 1m (sehr kurzfristig)
- `ATR1m_open` = Mittelwert von ATR1m_short über die ersten N Bars nach Open (z. B. N=5–10)
- `ATR1m_now` = aktueller ATR1m_short

**Decay-Kriterium (Beispiel):**
- `ATR1m_now / ATR1m_open < 0.6` für M aufeinanderfolgende Bars (z. B. M=3)

Wenn erfüllt:
- Event `OPEN_VOL_DECAYED` wird gesetzt.
- Setup A darf von „WAIT_CONSOLIDATION“ → „READY“ wechseln (sofern weitere Kriterien passen).

### 26.2 Parabolic Acceleration Detector (Normativ)

**Ziel:** Parabolische Anstiege erkennen, um Teilgewinne vor möglichem Drawdown zu sichern.

Komponenten:
- `ret_1` = log(Close_t / Close_{t-1})
- `slope_N` = Sum(ret_1) über N Bars (z. B. N=5)
- `accel_N` = slope_N - slope_{N} vor k Bars (z. B. k=5)
- `range_ratio` = (High-Low) / ATR(14) (1m oder 3m)
- `vol_ratio` = Volume / SMA(Volume, 20)

**Parabolic-Kriterium (Beispiel, OR-Kombination):**
- accel_N > accel_threshold AND vol_ratio > 1.5
- oder: 2+ Bars mit range_ratio > 2.0 in kurzer Folge
- oder: 3 Bars in Folge mit „higher closes“ + steigendem Volumen (climax build)

Wenn erfüllt:
- Event `PARABOLIC_ACCEL` (severity = HIGH) wird gesetzt.
- Arbiter bevorzugt SCALE_OUT (nicht sofort Voll-Exit) sofern Trend noch intakt ist.

### 26.3 Exhaustion / Blow-off Detector (Normativ)

**Ziel:** Blow-off Top / Erschöpfung erkennen (typisch vor Plateau/Dump).

Komponenten:
- Upper wick ratio: `(High - max(Open,Close)) / (High-Low)` 
- Close position: `(Close - Low) / (High-Low)`
- Volume climax: vol_ratio > X (z. B. >2.5)

**Exhaustion-Kriterium (Beispiel):**
- Upper wick ratio > 0.6 AND vol_ratio > 2.0
- oder: Preis macht neues High, Close aber deutlich darunter (failed continuation)

Wenn erfüllt:
- Event `EXHAUSTION_SIGNAL` (severity=CRITICAL) → Immediate Decision Cycle zulässig
- OMS tighten trail aggressiv
- SCALE_OUT Ladder triggert sofort (sofern Position > minQty)

### 26.4 Crash / Shock Detector (Normativ)

**Ziel:** echte Schockbewegungen erkennen und nicht durch Outlier-Filter entfernen.

Komponenten:
- Move% in kurzer Zeit
- vol_ratio

**Crash-Kriterium (Beispiel):**
- MoveDown > 5% in <=2 Bars AND vol_ratio > 3

Aktion:
- Event `SHOCK_DOWN` (CRITICAL)
- RiskPolicy: Neue Entries für X Minuten blockiert (shockLockout)
- Wenn Position offen: nur Safety (Stops/Exit), kein Add

### 26.5 Reclaim Detector (Normativ)

**Ziel:** nach Flush/Korrektur den Rücksprung in Up-Regime erkennen.

Kriterien (Beispiel):
- Close kreuzt über VWAP
- Close über EMA9/EMA21 Cluster (3m)
- und es gibt keine neuen Lows in den letzten K Bars
- optional: “Volume on reclaim” > “volume on pullback” (Proxy)

Event `RECLAIM_CONFIRMED` (MEDIUM/HIGH)

---

## 27. Regime-Fusion & Reliability-Weighting (Hybrid „auf Augenhöhe“)

### 27.1 Motivation

Wenn Quant und LLM beide Regime labeln, ist die Frage:
- wem vertraut man wann mehr?

Antwort: **dynamisch**, aber deterministisch — basierend auf historischer Kalibrierung.

### 27.2 Reliability Scores (Normativ)

Pro Modellkomponente wird ein Reliability Score gepflegt:
- `R_quant` ∈ [0,1]
- `R_llm` ∈ [0,1]

Update erfolgt EOD anhand Ground-Truth (Kap. 9.9):
- Treffer → Score steigt leicht
- Fehler → Score sinkt stärker (asymmetrisch, konservativ)

### 27.3 Fusion-Formel (Normativ)

Finale Regime-Confidence:
- `conf_final = normalize( R_quant * conf_quant + R_llm * conf_llm )`

Finales Label:
- wenn Labels gleich: trivial
- wenn Labels verschieden:
  - wähle Label mit höherem gewichteten Confidence
  - aber: Confirmation10m kann DOWN als Hard Gate setzen

### 27.4 Safety Override (Normativ)

Unabhängig von Reliability:
- DOWN confirmed (10m) blockiert neue Entries.
- DQ risk blockiert neue Entries.

---

## 28. Parabolic Profit-Taking als deterministische Ladder

### 28.1 Ladder-Profile (Normativ)

Das System kennt Profile (bounded enums), die Quant baseline setzt und LLM nur als Bias wählen darf:

- `SCALE_OUT_PROFILE = FAST | STANDARD | SLOW`

Beispiel STANDARD:
- bei +1R: sell 25%
- bei +2R: sell 25%
- bei PARABOLIC_ACCEL: immediate sell 20%
- Runner: rest trail

FAST:
- früher & mehr (für HIGH_VOL / blow-off risk)

SLOW:
- weniger früh (für steady trend days)

### 28.2 Trigger-Kombination (Normativ)

Scale-Out wird ausgelöst durch:
- R-Multiple thresholds (deterministisch)
- oder Parabolic/Exhaustion Events

Conflict Handling:
- wenn LLM EXIT_BIAS bei CRITICAL, Quant aber Trend intakt → SCALE_OUT FAST statt EXIT.

---

## 29. Re-Entry & Aufstocken: „Nicht in fallendes Messer“

### 29.1 Re-Entry Lockouts (Normativ)

- Nach Exit: Cooldown
- Nach SHOCK_DOWN: shockLockout länger
- Nach 3 Verlusten in Folge: globaler Cooling-Off (z. B. 30m)

### 29.2 Re-Entry Gate (Normativ)

Re-Entry erfordert:
- `RECLAIM_CONFIRMED`
- Higher-Low bestätigt (Swing)
- Confirmation10m nicht DOWN
- Quant score >= threshold
- LLM nicht CAUTION/VETO (oder CAUTION nur wenn urgency HIGH + quant sehr stark)

### 29.3 Add Gate (Normativ)

Add nur wenn:
- initial position im Gewinn oder zumindest nicht strukturell invalidiert
- kein neues Low seit Entry
- Trail bereits auf > initial stop (Risiko reduziert)

---

## 30. Backtester-Architektur (Engineering-konzeptionell)

### 30.1 Event-driven Replay (Normativ)

Backtester simuliert exakt die gleichen Events wie Live:
- Bar1mReceived → Resample → Decision Cycles → Orders → Fills → Stops

Wichtig:
- Decision Loop läuft auf gleichen Komponenten (QuantEngine/Arbiter/Risk/OMS).
- LLM wird über CachedAnalyst ersetzt.

### 30.2 Determinismus-Guarantee (Normativ)

- Keine RNG ohne Seed.
- MarketClock im Backtest deterministisch.
- LLM Outputs aus Cache, nicht „live“.

### 30.3 Output-Artefakte

- Equity Curve (optional)
- Trade List
- Event Timeline
- Challenger Pass/Fail

---

## 31. Reporting & Analytics (EOD) (Normativ)

EOD Report enthält:

- Trades (Entry/Exit Zeit, Setup, Cycle)
- P&L: realized/unrealized
- MAE/MFE
- Slippage proxy
- Regime calibration update (GT vs pred)
- LLM/Quant agreement matrix
- DQ/Incidents summary
- Parameter snapshot + versions

---


## 32. Teststrategie (Unit/Integration/Property)

### 32.1 Unit Tests (Normativ)

- Indikatoren: EMA/RSI/ATR/ADX auf bekannten Testreihen
- Resampling: 1m→3m/10m Aggregation korrekt (open/high/low/close/volume)
- DQ Gates: Reihenfolge korrekt (Crash-Markierung vor Outlier-Reject)
- Arbiter Tables: jede Kombination QuantVote/LLMBias → erwarteter Intent

### 32.2 Integration Tests (Normativ)

- Replay eines Tages mit deterministischem Ergebnis (gleiche Events, gleiche Trades)
- OMS: Partial fills + Stop/Target Updates
- Degradation: LLM timeout → Quant-only + no-new-entry
- Kill-Switch: end-to-end (Orders cancel + Position close)

### 32.3 Property Tests (Geplant)

- Invarianten:
  - Position > 0 ⇒ es existiert ein Schutzmechanismus
  - Stops werden nie gelockert
  - EOD ⇒ Position == 0
  - EventLog Hash-Chain valid

---

## 33. Data Storage & Retention (konzeptionell)

### 33.1 Speicherobjekte

- Raw Bars (1m) — für Replay/Forensik
- Resampled Bars (3m/10m) — optional, ableitbar
- EventLog (append-only) — zentral
- ConfigSnapshots (Versionen, Parameter)
- Reports (EOD)

### 33.2 Retention Policy (Geplant)

- Events & Reports: 2–5 Jahre (je nach Storage)
- Raw Bars: abhängig von Lizenz, mindestens 6–12 Monate
- LLM Cache: mindestens solange, wie Backtests reproduziert werden müssen

---

## 34. Performance- und Robustheitsanforderungen

### 34.1 Latenzbudgets (Geplant)

Da 1m Bars die kleinste Zeiteinheit sind, ist das System nicht ultra-low-latency, aber:

- Decision Cycle (3m close → intent) < 500ms typisch
- OMS updates (stop tighten auf 1m close) < 200ms typisch
- LLM Calls dürfen langsamer sein, aber TTL muss respektiert werden

### 34.2 Fail-fast Prinzip (Normativ)

Bei widersprüchlichen Daten/States:
- lieber **NO_ACTION** als falscher Trade
- lieber **HALT** als unkontrolliertes Weiterhandeln

---

## 35. Implementierungsreihenfolge (MVP Plan)

1. Data Layer + MarketClock + EventLog
2. Resampling + KPI Engine + Warmup
3. QuantEngine + Arbiter (ohne LLM)
4. OMS/Execution Simulator + Backtester
5. Challenger-Suite (synthetisch)
6. LLM Integration (schema + cache) + Degradation
7. Paper Trading + Reports + Dashboard
8. Klein-Live Rollout mit Release-Gates

---


## 36. Risk Gate: explizite Checks (Pre-Trade & In-Trade)

### 36.1 Pre-Trade Checks für ENTER/ADD (Normativ)

Reihenfolge (fail-fast):

1. **Session Gate**
   - innerhalb erlaubter Phase?
   - entry_cutoff respektiert?
2. **DQ Gate**
   - keine CRITICAL DQ Events aktiv?
3. **Daily Budget**
   - effective_budget > 0?
4. **Trade Count / Cycle Limits**
   - max_round_trips nicht überschritten?
   - max_cycles nicht überschritten?
5. **Stop Feasibility**
   - stop_dist innerhalb bounds?
   - nicht zu eng (Noise) / nicht zu weit (unwirtschaftlich)
6. **R/R Mindestkriterium**
   - erwartetes 1. Target erreichbar (z. B. min 1R)
7. **Liquidity Proxy**
   - Volume >= min
   - Range/ATR nicht extrem (illiquide spikes)
8. **Concentration**
   - max Position Value nicht überschritten
9. **Shock Lockouts**
   - nach SHOCK_DOWN: entry block für X Minuten

### 36.2 In-Trade Checks (Normativ)

1. **Stop vorhanden?**
   - falls nein: EMERGENCY, sofort exit policy
2. **Stop Tighten Only**
3. **Risk reduzieren bei Uncertainty**
   - wenn DQ warnings oder LLM/Quant stark widersprechen: SCALE_OUT bevorzugen
4. **EOD Regeln**
   - forced close strikt

### 36.3 Post-Trade Checks (Normativ)

- realized loss update
- cooldown setzen (abhängig von Ergebnis)
- cycle summary event
- wenn daily limit erreicht → DAY_STOPPED

---

## 37. Konfiguration, Feature Flags, Experimentierung

### 37.1 Feature Flags (Normativ)

Damit Backtests/Paper/Live vergleichbar bleiben:

- Flags müssen im ConfigSnapshot stehen:
  - enable_llm (true/false)
  - enable_immediate_decisions (true/false)
  - enable_setup_C (true/false)
  - cost_model_variant (id)

### 37.2 Parameter-Tuning Leitlinien (Normativ)

- Tuning nur über Walk-Forward
- Parameter nicht pro Tag adaptiv „lernen“ (sonst Leakage-Risiko)
- Änderungen nur via semver + Regression Gate

---

## 38. Alerting: konkrete Schwellen (Beispiele)

### 38.1 Data

- `DQ_STALE_FEED` WARNING ab 90s, CRITICAL ab 180s
- `DQ_OUTLIER_REJECTED` > 3 mal/Tag → Investigate

### 38.2 LLM

- invalid schema rate > 0% → block release
- timeout rate > 2% intraday → degrade + incident

### 38.3 Execution/Broker

- reject rate > 0.5% → incident
- cannot liquidate → EMERGENCY

---


## 39. Setup-Rulebooks (detailliert, deterministic)

Dieses Kapitel spezifiziert die **Gate-Reihenfolge** und typische **ReasonCodes** je Setup.
Ziel: Implementierer können identische Logik bauen, und Logs sind auswertbar.

### 39.1 Gemeinsame Gate-Reihenfolge (Normativ)

Für jedes Setup gilt:

1. Session Gate
2. DQ Gate
3. Regime Gate (10m Confirmation)
4. Setup Pattern Gate (deterministisch)
5. Quant Score Gate
6. Hybrid Gate (Dual-Key)
7. Risk Gate
8. Execution Feasibility (Order policy)

### 39.2 Setup A — Gate Details

**A1: Session Gate**
- block, wenn innerhalb OpeningBlock
  - reason: `SESSION_OPENING_BLOCK`
- block, wenn nach entry_cutoff
  - reason: `SESSION_ENTRY_CUTOFF`

**A2: Pattern Gate (Consolidation)**
- benötigt `OPEN_VOL_DECAYED` Event ODER RangeCompressionFlag
- RangeCompressionFlag:
  - avgRange(last 3 bars) < avgRange(opening baseline) * 0.7
- kein neues Low in last K bars (z. B. K=5)

Fail reasons:
- `A_NO_VOL_DECAY`
- `A_NO_RANGE_COMPRESSION`
- `A_NEW_LOW_DURING_CONSOLIDATION`

**A3: Structure Gate**
- Close(3m) über VWAP (oder reclaim innerhalb letzter 2 Bars)
- EMA9 >= EMA21 (tolerant), slope(EMA9) positiv

Fail reasons:
- `A_BELOW_VWAP`
- `A_EMA_NOT_ALIGNED`

**A4: Overextension Gate**
- DistToVWAP_ATR <= maxExtension (z. B. 1.5)
- RSI <= RSI_max (z. B. 75) außer Momentum-Mode

Fail reasons:
- `A_OVEREXTENDED`
- `A_RSI_OVERHEAT`

**A5: Quant Score Gate**
- quant_score >= threshold
Fail reason:
- `A_SCORE_TOO_LOW`

**A6: Hybrid Gate**
- LLM bias not VETO_ENTRY
Fail reason:
- `LLM_VETO_ENTRY`

### 39.3 Setup B — Gate Details (Flush → Reclaim)

**B1: Flush Gate**
- `FLUSH_DETECTED` Event muss existieren (innerhalb last X Minuten)
Fail reason:
- `B_NO_FLUSH_EVENT`

**B2: Knife-Catching Block**
- solange neue Lows entstehen: block
Fail reason:
- `B_NEW_LOW_CONTINUES`

**B3: Reclaim Gate**
- Close(3m) > VWAP
- plus: Close > EMA9 und EMA9 steigt

Fail reasons:
- `B_NO_RECLAIM_VWAP`
- `B_NO_RECLAIM_MA`

**B4: Starter Entry vs Confirmation**
- StarterEntry erlaubt nur mit kleiner Größe (policy)
- Add erst, wenn:
  - Higher-Low bestätigt
  - 10m Confirmation nicht DOWN

Fail reasons:
- `B_CONFIRMATION_MISSING`

### 39.4 Setup C — Gate Details (Coil → Breakout)

**C1: Coil Detection**
- ATR sinkt über Fenster
- Swings konvergieren (Highs fallen, Lows steigen)
Fail reason:
- `C_NO_COIL`

**C2: Breakout Gate**
- Close(3m) > coilHigh + buffer
- VolumeRatio > threshold

Fail reasons:
- `C_BREAKOUT_WEAK`
- `C_VOLUME_NOT_CONFIRMED`

**C3: Fakeout Risk Handling**
- Wenn LLM red_flag FAKEOUT_RISK:
  - bevorzugt Retest Entry (State C_RETEST_OPTIONAL)
  - wenn kein Retest innerhalb N Bars: abort
Fail reason:
- `C_NO_RETEST_ABORT`

### 39.5 Setup D — Gate Details (Re-Entry/Scale-In)

**D1: Eligibility**
- cycleNumber >= 2 ODER (vormittags kein Trade, aber trend switch)
Fail reason:
- `D_NOT_ELIGIBLE`

**D2: Cooldown**
- cooldown abgelaufen
Fail reason:
- `D_COOLDOWN_ACTIVE`

**D3: Bottoming + Reclaim**
- Reclaim confirmed + Higher-Low
Fail reasons:
- `D_NO_RECLAIM`
- `D_NO_HIGHER_LOW`

**D4: Conservative Risk**
- sizing reduziert
- ProfitProtection EARLY
Fail reason:
- `D_RISK_POLICY_BLOCK`

---

## 40. ReasonCode-Design: Taxonomie (Empfehlung)

ReasonCodes sollten maschinenlesbar und hierarchisch sein:

- `SESSION_*`
- `DQ_*`
- `RISK_*`
- `LLM_*`
- `EXEC_*`
- `A_*`, `B_*`, `C_*`, `D_*`

Dadurch kann man später:
- Heatmaps bauen (warum werden Trades geblockt?)
- Fehlkalibrierung erkennen (zu streng vs zu lax)
- LLM-Veto-Qualität messen

---


## 41. Appendix D — Mathematische Indikator-Definitionen (kurz, aber eindeutig)

### 41.1 EMA

EMA mit Periode p auf Serie x_t:

- `alpha = 2 / (p + 1)`
- `EMA_t = alpha * x_t + (1 - alpha) * EMA_{t-1}`

Initialisierung:
- `EMA_0` = SMA der ersten p Werte (oder erster Wert; konsistent über Backtests!)

### 41.2 RSI (14)

Standard RSI auf Close:

- `U_t = max(C_t - C_{t-1}, 0)`
- `D_t = max(C_{t-1} - C_t, 0)`
- Wilder's smoothing für AvgU/AvgD
- `RS = AvgU / AvgD`
- `RSI = 100 - 100 / (1 + RS)`

### 41.3 True Range & ATR (14)

- `TR_t = max(High_t - Low_t, abs(High_t - Close_{t-1}), abs(Low_t - Close_{t-1}))`
- ATR über Wilder smoothing auf TR.

### 41.4 ADX (14)

- +DM/-DM nach Wilder
- +DI/-DI = 100 * smoothedDM / ATR
- DX = 100 * abs(+DI - -DI) / (+DI + -DI)
- ADX = Wilder smoothing auf DX

> Diese Definitionen sind wichtig, damit Backtest und Live exakt gleich sind.

---

## 42. Appendix E — Slippage-/Kostenmodell (Proxy) als deterministische Funktion

Da bid/ask fehlt, muss Slippage als Proxy modelliert werden. Wichtig ist Konsistenz, nicht „Perfektion“.

### 42.1 Baseline Slippage

- `slippage_ticks = base_ticks`
- `slippage_price = slippage_ticks * tick_size` (tick_size konfigurierbar oder implizit)

### 42.2 Volatility-abhängiger Anteil

- `vol_factor = clamp( range_ratio / target_range_ratio, 0.5, 3.0 )`
- `liq_factor = clamp( 1 / max(volume_ratio, 0.5), 0.5, 3.0 )`
- `slippage = baseline + k1 * ATR * vol_factor * liq_factor`

### 42.3 Shock Slippage

Wenn Event `SHOCK_DOWN` oder `PARABOLIC_ACCEL`:
- `slippage *= shock_multiplier` (z. B. 2.0)

### 42.4 Konservativität

Backtests sollten eher konservativ sein:
- bei Buy: slippage erhöht Entry-Preis
- bei Sell: slippage senkt Exit-Preis

---


## 43. Day-Type Klassifikation & Strategy Switching (Regimebeurteilung)

### 43.1 Day Types (Normativ)

Neben lokalen Regime-Labels führt das System Day-Type Flags:

- `TREND_DAY_UP`
- `TREND_DAY_DOWN` (in Long-only relevant als Block/Exit)
- `RANGE_DAY`
- `HIGH_VOL_DAY`

Diese Flags werden primär aus 10m Confirmation + Volatilitätsmaßen erzeugt.

### 43.2 Deterministische Day-Type Heuristik (Beispiel)

- TREND_DAY_UP wenn:
  - Preis für >= 3 10m-Bars über VWAP
  - EMA(20) 10m steigt
  - ADX(14) 10m > threshold
- RANGE_DAY wenn:
  - ADX niedrig
  - häufige VWAP-Crossings
- HIGH_VOL_DAY wenn:
  - Range/ATR im 10m Frame über Schwelle

### 43.3 Strategy Switching Regeln (Normativ)

Switching ist nicht frei; es ist gated:

- Wenn RANGE_DAY: Setup A/C (Breakout) restriktiv, Setup B/D (Reclaim) bevorzugt.
- Wenn TREND_DAY_UP: Setup A/C bevorzugt, Setup B nur nach klarer Reclaim-Struktur.
- Wenn HIGH_VOL_DAY: sizing reduziert, Profit Protection früher, FAST scale-out profile.

LLM darf Switching empfehlen, aber Arbiter muss es mit Confirmation/Heuristik konsistent halten.

---

## 44. Setups × Regime Matrix (Normativ)

| Regime/DayType | Setup A | Setup B | Setup C | Setup D |
|---|---:|---:|---:|---:|
| TREND_DAY_UP | ✓✓ | ✓ | ✓✓ | ✓ |
| RANGE_DAY | ✓ (konservativ) | ✓✓ | ✓ (nur stark) | ✓✓ |
| HIGH_VOL_DAY | ✓ (klein) | ✓ (klein) | ✓ (klein) | ✓ (klein) |
| DOWN confirmed | ✗ | (nur nach Reclaim) | ✗ | (nur nach Trendwechsel) |

Legende:
- ✓✓ = bevorzugt
- ✓ = erlaubt, aber restriktiv
- ✗ = blockiert

---

## 45. Noise vs. Reaktionsgeschwindigkeit: 1m→3m→10m (Begründung)

### 45.1 Warum 1m Rohsignal + 3m Decision?

- 1m ist nötig, um:
  - Crash/Spike, Exhaustion, Parabolic früh zu erkennen,
  - Stops/Tightening schnell zu reagieren.
- 3m reduziert Noise für:
  - EMA/RSI/ADX Signale,
  - Entry/Exit Entscheidungen, die sonst „flattern“ würden.

### 45.2 Warum 10m Confirmation?

- Trend-/Regime-Confirmation braucht Stabilität.
- 10m dient als Anti-Flip Filter: schützt vor „VWAP-Magnet Chop“.

### 45.3 Extrem-Situationen

- Bei CRITICAL Events darf 1m Monitoring einen Immediate Decision Cycle auslösen,
  ohne das 3m Raster zu verlassen — aber nur für Safety (Scale-out/Exit), nicht für hektische Entries.

---


## 46. End-to-End Beispiele (Timeline-basiert)

Diese Beispiele sind keine Backtest-Daten, sondern **Blueprints**, wie das System reagieren soll.

### 46.1 Beispiel E1 — Opening Spike → Konsolidierung → Trend → Parabolic → Dump (Plateaubildung)

**09:30–09:35**
- 1m Bars: Range sehr groß, VolumeRatio hoch
- Events:
  - `OPENING_SPIKE` (INFO/HIGH)
- State:
  - `WARMUP` → `OBSERVING` (Entry weiterhin blockiert durch Opening-Block)

**09:35–09:45**
- Range nimmt ab, VWAP stabilisiert
- Events:
  - `OPEN_VOL_DECAYED`
  - `CONSOLIDATION_FLAG`
- State:
  - `SEEKING_ENTRY` (wenn Warmup fertig)

**09:45 (3m close)**
- Quant:
  - regime UP, score 0.62, intent ENTER
- LLM:
  - ALLOW_ENTRY, urgency MEDIUM, keine redFlags
- Arbiter:
  - ENTER approved
- OMS:
  - Limit-Order in Entry-Zone, nach Fill bracket: Stop + Targets

**10:10**
- Preis beschleunigt, Range/ATR steigt, Volumen steigt
- 1m Events:
  - `PARABOLIC_ACCEL`
- Immediate Decision Cycle:
  - Arbiter entscheidet SCALE_OUT (Profile STANDARD/FAST)
- OMS:
  - 20% sofort, trail enger

**10:30–12:00**
- Plateau: Momentum nimmt ab, Dochte oben, Volumen climax
- 1m Events:
  - `EXHAUSTION_SIGNAL` (CRITICAL)
- Arbiter:
  - SCALE_OUT (weitere Tranche) oder EXIT für Rest (konservativ, abhängig von Trendfilter)

**15:45**
- Forced Close Phase
- OMS:
  - Rest schließen falls noch offen → EOD-flat

Pass/Fail-Kriterium:
- Entry nicht im Opening-Block
- Scale-out vor Dump
- EOD-flat immer

### 46.2 Beispiel E2 — Opening Flush → Reclaim → Run → Re-Entry nach Pullback

**09:30–09:35**
- -8% Abverkauf, vol_ratio extrem
- Events:
  - `SHOCK_DOWN` / `FLUSH_DETECTED`
- Policy:
  - shockLockout aktiv (keine Entries während Fall)

**09:40–10:00**
- Stabilisierungsphase, kein neues Low
- 3m: Close reclaim’t VWAP
- Events:
  - `RECLAIM_CONFIRMED`
- Quant: score ausreichend, intent ENTER (Starter)
- LLM: CAUTION aber urgency HIGH (V-Reversal), keine fakeout redFlag
- Arbiter: erlaubt Starter-Entry (klein)

**10:20**
- Higher-Low bestätigt, 10m Confirmation nicht DOWN
- Arbiter: ADD erlaubt (Sizing reduziert)
- Management: Profit Protection früh

**11:30**
- Teilgewinn realisiert (2R), Exit eines Teils, COOLDOWN für neue Entries
- Danach Pullback bis nahe VWAP, dann erneuter Reclaim
- Setup D wird eligible (cycleNumber=2)
- Re-Entry klein, schnellere Profit Protection

Pass/Fail:
- Kein Averaging down im Flush
- Add nur nach Confirmation
- Cycle2 sizing reduziert

### 46.3 Beispiel E3 — Datafeed Stall / indirekter Trading Halt

**13:05**
- keine neuen Bars für > dq_stale_seconds_warn
- Alert WARNING

**13:07**
- > dq_stale_seconds_halt
- `DATA_HALT`, State → DAY_STOPPED (oder spezieller HALT-State)
- keine neuen Orders

**13:12**
- Feed kommt zurück
- System bleibt im Safe Mode bis manuell resume (oder automatische „stabil“ Regel)

Pass/Fail:
- keine Trades während stale feed
- Audit enthält DQ Events

---


## 47. Appendix F — Snapshot- & DecisionLog-Schemata (Fachformat)

### 47.1 MarketSnapshot (Normativ)

Ein MarketSnapshot ist der minimale Input für Brain/Risk und MUSS versioniert werden.

**Felder (Beispiel):**
- `runId`
- `instrumentId`
- `timestamp` (MarketClock)
- `sessionPhase`
- `bar1m` (aktuelles 1m OHLCV)
- `bars1m_lastN` (optional, komprimiert)
- `bar3m_lastClosed` (OHLCV)
- `bar10m_lastClosed` (OHLCV)
- `vwap_day`
- `levels`:
  - prior_close
  - prior_high/low
  - day_open
  - day_high/low so far
- `dq_state`:
  - staleFeedFlag
  - lastDqEvent
- `monitor_events_active` (Liste)

### 47.2 FeatureSnapshot (Normativ)

FeatureSnapshot ist frame-spezifisch.

**FeatureSnapshot3m (Beispiel):**
- `ema9`, `ema21`
- `rsi14`
- `atr14`
- `adx14`
- `volume_ratio`
- `range_atr_ratio`
- `dist_to_vwap_atr`
- `pattern_flags` (coil, reclaim, etc.)

FeatureSnapshots enthalten zusätzlich:
- `feature_version`
- `calc_window_info`

### 47.3 DecisionLog (Normativ)

Jede 3m-Entscheidung erzeugt genau einen DecisionLog.

**Felder:**
- `decisionId`, `runId`, `cycleNumber`, `timestamp`
- `state_before`, `state_after`
- `market_snapshot_hash`
- `feature_snapshot_hash`
- `quant_vote` (vollständig)
- `llm_vote` (vollständig oder „missing“ + reason)
- `arbiter_intent` + `arbiter_params`
- `risk_gate_result` (pass/fail + reasons)
- `execution_commands` (order intents)
- `outcome_link` (wird später gefüllt: fill/exit results)

### 47.4 LLMCall Log (Normativ)

**Felder:**
- `llmCallId`, `decisionId` (optional), `timestamp`
- `prompt_version`, `schema_version`
- `input_hash`, `token_count_estimate`
- `latency_ms`, `timeout_flag`
- `output_json` (raw) + `validation_result`
- `ttl_seconds` (final)

---


## 48. Operational Runbook (MVP)

### 48.1 Start-of-Day Checklist (Normativ)

1. System Health:
   - Datafeed erreichbar (Heartbeat ok)
   - Broker erreichbar (auth ok)
   - LLM erreichbar (optional, aber Status bekannt)
2. Config Freeze:
   - Watchlist/Instrumente festlegen
   - configSnapshot erzeugen (hash)
3. Simulation/Paper/Live Mode prüfen:
   - falscher Mode = Start verweigern
4. Kill-Switch Test (dry-run):
   - Control-Channel erreichbar

### 48.2 Intraday Incident Handling (Normativ)

**DQ_STALE_FEED**
- wenn WARNING: beobachten, keine neuen Entries
- wenn CRITICAL: DATA_HALT, ggf. risk-reducing exit

**LLM Outage**
- QUANT_ONLY
- keine neuen Entries, bestehende Position managen

**Broker Reject**
- CRITICAL Alert
- automatische Retries begrenzt
- wenn Position ungeschützt: sofort risk-reducing exit (fallback)

### 48.3 End-of-Day Checklist (Normativ)

1. EOD-flat verifizieren (Position=0)
2. Reconciliation:
   - orders/fills konsistent?
3. Reports:
   - RunSummaryEod erzeugen
4. Archive:
   - EventLog hash-chain verifizieren

---


## 49. Offene Punkte (bewusst dokumentiert)

Ein Konzept ist auch dann „vollständig“, wenn offene Entscheidungen transparent sind.

### 49.1 Parameter-Kalibrierung

- exakte Schwellen (z. B. crash_move_pct, volume_ratio thresholds) müssen per Datenanalyse/Backtests kalibriert werden
- Ziele:
  - Crash vs Outlier sauber trennen
  - Overtrade in RANGE reduzieren

### 49.2 Session-Model

- Pre-/Post-Market handeln oder nur RTH?
- falls Pre: andere DQ/Illiquiditäts-Gates

### 49.3 Universe/Scanner (Out of Scope, aber Schnittstelle nötig)

- Watchlist kommt extern; Schnittstelle definieren:
  - daily selection file / API / UI input

### 49.4 Corporate Actions Handling

- Splits/Dividenden: adjustierte Historie oder Ausschluss
- Muss konsistent zwischen Backtest und Live sein

### 49.5 Broker-seitige Stop-Unterstützung

- Wenn Broker bracket/OCO unterstützt, nutzen.
- Wenn nicht, interner Stop nur mit extrem hoher Zuverlässigkeit (Watchdog) betreiben.

---

## 50. Appendix H — LLM Negative Tests (Safety Contract)

Diese Tests stellen sicher, dass LLM nicht „aus Versehen“ Trading-Details liefert.

### 50.1 Verbotene Outputs (Normativ)

LLM Output ist FAIL, wenn er:
- konkrete Stückzahlen nennt („buy 100 shares“)
- konkrete Preise/Stops nennt („stop at 101.23“)
- Ordertypen erzwingt („use market order“)
- nicht im JSON Schema ist

### 50.2 Prompt Injection Testfälle (Geplant)

Wenn optionale externe Inputs integriert werden (z. B. News-Metadaten), dann:
- Inputs werden als untrusted markiert
- LLM soll nicht auf direkte Anweisungen reagieren („ignore previous rules“)

---


## 51. Designprinzipien (Kurzform)

- **Fail-safe statt fail-open:** im Zweifel NO_ACTION oder HALT.
- **Determinismus in der Orchestrierung:** Arbiter/Risk/OMS deterministisch; LLM bounded.
- **Reproduzierbarkeit:** jedes Signal/Decision ist replaybar (Events + LLM cache).
- **Separation of Concerns:** Daten, Brain, Risk/Execution, Observability getrennt.
- **No silent changes:** jede Parameter-/Prompt-Änderung ist versioniert und gated.
- **Safety first:** EOD-flat, Kill-Switch, Hard-Stop sind nicht verhandelbar.

---

