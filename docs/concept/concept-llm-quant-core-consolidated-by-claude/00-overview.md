# ODIN -- Konsolidiertes Zielbild-Konzept

## 00 -- Systemuebersicht, Scope, Constraints, Designphilosophie

**Version:** Konsolidiert (Stand: 2026-02-21)
**Konsolidiert aus:** Fachkonzept v1.5, Strategie-Optimierung-Sparring, Master-Konzept v1.1, Unified Concept v2.1, Build-Spec v3.0, Evaluationsdesign, Testplan, Stakeholder-Feedback 1+2, L2-Kompromiss

> **Normative Sprache:** MUSS = zwingend erforderlich, DARF NICHT / NIE = verboten, SOLL = dringend empfohlen (Abweichung nur mit dokumentierter Begruendung), KANN = optional.

---

## 1. Was ist ODIN?

**ODIN** (Orderflow Detection & Inference Node) ist ein vollautomatischer Intraday-Trading-Agent fuer Aktien (US, Europa). Das System kombiniert drei Saeulen:

1. **LLM-Situationsanalyse** -- Regime-Erkennung, Kontextsignale, taktische Parametersteuerung
2. **Deterministische Regellogik** -- Entry/Exit-Entscheidungen, Pattern-State-Machines, Quant-Scoring
3. **Quantitative Validierung** -- Indikatoren, Scoring-Modell, Veto-Mechanismus

Das System betreibt 1--3 isolierte Pipelines parallel. Jede Pipeline handelt genau ein Instrument pro Tag. Die Instrumente werden extern vorgegeben und beim Start der Pre-Market-Phase (07:00 ET) eingefroren -- kein Symbolwechsel waehrend des Handelstages. Am Ende des Handelstages MUESSEN alle Positionen aller Pipelines geschlossen sein (EOD-Flat).

---

## 2. Systemparameter und Rahmenbedingungen

| Parameter | Wert | Anmerkung |
|-----------|------|-----------|
| Tageskapital | 10.000 EUR | Aufgeteilt auf aktive Pipelines |
| Max. Tagesverlust (Hard-Stop) | -10% (1.000 EUR) | Global ueber alle Pipelines und alle Zyklen |
| Instrumente | 1--3 parallel | Je Pipeline genau ein Instrument |
| Richtung | Long only | Kein Short |
| Instrumenttyp | Aktien | Keine Derivate |
| Datenbasis | 1-Min OHLCV + Volume | Kein Orderbuch fuer Alpha/Regime/Setups (OHLCV-only Constraint) |
| Rohsignal | 1m | Monitoring-Ebene |
| Quant-Decision-Frame | 3m | Resampled, weniger Noise |
| Confirmation-Frame | 10m | Trend-/Regime-Hysterese, Fakeout-Filter |
| Handelszeiten | Pre-Market + RTH | Exchange-konfigurierbar |
| Autonomie | Vollautomatisch | Mit manuellen Controls (Pause/Kill) |
| EOD-Flat | MUSS | Keine Overnight-Position |
| Max. Zyklen pro Tag pro Pipeline | 3 (konfigurierbar) | Hard-Cap als Guardrail |
| Max. Round-Trip-Trades/Tag | 5 | Global ueber alle Pipelines und Zyklen |

**Instrumentauswahl:**
Instrumente werden extern vorgegeben (Scanner/Watchlist ausserhalb des Systems). Die Auswahl wird zu Session-Start eingefroren. Wird ein Instrument vor RTH gehalted oder mit einer Trading-Restriction belegt, wechselt die betroffene Pipeline in DAY_STOPPED (kein Ersatzsymbol, andere Pipelines laufen weiter).

**Anforderungen an geeignete Instrumente:**

- Aktien (US, Europa)
- Ausreichende Liquiditaet (Spread < 0.5%, taegl. Volumen > 500.000)
- Keine Penny Stocks (Preis > 5 USD/EUR)
- Handelbar ueber IB TWS

---

## 3. Scope und Nicht-Ziele

### In Scope

- Intraday Long-Trades auf Aktien
- Ein- und Ausstieg, Teilverkaeufe, Trailing-Stops, Re-Entry/Scale-In
- Multi-Cycle-Handling (mehrere Episoden pro Tag pro Instrument)
- Degradation Modes (LLM-Ausfall, Daten-Feed-Ausfall)
- Backtesting + Walk-Forward + Paper/Klein-Live Pipeline
- Evaluationsdesign (MC vs. Unified, FusionLab)

### Nicht-Ziele / Out-of-Scope

- Short-Selling
- Optionen, Futures, Derivate
- Portfolio-Optimierung / Korrelationen ueber viele Instrumente
- HFT / Sekunden-Scalping
- Orderbuch-/Spread-Arbitrage
- News-getriebene Strategie (News hoechstens optional als strukturierte Metadaten)
- Multi-Broker-Support
- Cloud-Deployment
- Mobile App
- Vision-Modul (Chart-Bild-Analyse) -- erst nach erfolgreicher Validierung des zahlenbasierten Kerns

---

## 4. Harte Constraints

### 4.1 OHLCV-Only Constraint (Stakeholder-Entscheidung: L2-Kompromiss)

Alpha-Generierung, Regime-Erkennung und Setup-Aktivierung basieren ausschliesslich auf **1-Minuten OHLCV-Daten** (Open/High/Low/Close/Volume). Nicht verfuegbar fuer diese Zwecke: Bid/Ask, Spread, Orderbuch (L2), Ticks, Marktbreite, Imbalance.

**Konsequenzen:**

- Liquiditaets- und Ausfuehrungsqualitaet wird ueber Proxies modelliert (Volumen/Range/Fill-Rate)
- Alle Signale MUESSEN aus OHLCV ableitbar sein
- Backtest: konservatives Kosten-/Fill-Modell + Stress-Szenarien

**L2-Execution-Enhancement (optional, nur Live):**
L2-Orderbuchdaten KOENNEN optional im Live-Betrieb fuer **Execution-Verbesserung** verwendet werden (bessere Fill-Qualitaet, Spread-Monitoring). L2 ist NICHT backtest-kritisch und beeinflusst KEINE Entscheidungslogik (Alpha/Regime/Setups). Diese Trennung stellt sicher, dass alle Backtest-Ergebnisse ohne L2 reproduzierbar sind.

### 4.2 Zeitrestriktion

- Alle Zeiten beziehen sich auf die Boersen-Zeitzone des Instruments (`America/New_York` fuer US)
- **MarketClock** ist die einzige Zeitquelle im Trading-Codepfad. Kein `Instant.now()` in Decision-/Risk-/OMS-Logik
- Trading-Kalender MUSS Feiertage, halbe Handelstage, DST-Umstellungen korrekt abbilden

### 4.3 Namespace

Alle Konfigurationen verwenden den Namespace `odin.*` (nicht `moses.*`).

---

## 5. Designphilosophie

### 5.1 Kernprinzip: Symmetrischer Hybrid (Stakeholder-Entscheidung)

ODIN ist KEIN reines Algo-Trading-System. Das LLM MUSS unterstuetzende Entscheidungsgewalt haben -- aber gebunden an deterministische Leitplanken. Das System ist ein **symmetrischer Hybrid**:

- **QuantEngine**: deterministische KPIs, Signale, Scores, Vetos, Risk-Fakten
- **LLM**: Lagebeurteilung (Regime/Szenario), Pattern-Hypothesen, taktische Profile/Parameter (bounded), Kontext-Interpretation

**Gleichberechtigung bedeutet:**

- Beide liefern **Vote + Confidence** (Entry/Exit/No-Trade)
- Beide koennen **Soft-Vetos** setzen (z.B. "Unsicherheit", "Exhaustion Risk")
- Der Arbiter kombiniert beides **deterministisch**
- **Ausnahme:** Harte Risk-Exits (Stop, Forced Close, Kill-Switch) sind nicht verhandelbar

### 5.2 Gegenseitige Schutzwirkung

**Quant schuetzt vor LLM-Fehlern:**

- Invalides Schema wird verworfen
- LLM-Vote ohne Quant-Bestaetigung fuehrt zu keinem Entry
- Numerische Grenzen: keine Stop/Size-Freiheit fuer das LLM

**LLM schuetzt vor Quant-Blindheit:**

- Erkennt "Plateau ist Distribution" vs. "Plateau ist Pause"
- Erkennt Wechsel in "HighVol/Chop" frueher, bevor Quant-Filter zu spaet reagieren
- Kann "No-Trade" empfehlen, obwohl Quant formal ok ist (z.B. unsaubere Struktur)

### 5.3 Asymmetrische Verantwortung (innerhalb des symmetrischen Modells)

| Faehigkeit | Verantwortlich | Begruendung |
|------------|---------------|-------------|
| Marktlage diagnostizieren (Regime) | QuantEngine + LLM (Fusion) | KPI primaer, LLM sekundaer -- empirisch zu kalibrieren |
| Kontextsignale und Opportunity Zones liefern | LLM | Kontextuelles Verstaendnis, das Quant nicht leisten kann |
| Entry/Exit-Entscheidung treffen | Arbiter (deterministisch) | Testbar, reproduzierbar, kein stochastisches Verhalten |
| Entry-Signale validieren | QuantEngine | Deterministische Gegenkontrolle, Anti-Halluzination |
| Position Sizing | Risk Gate | MUSS deterministisch und verlaesslich sein |
| Stop-Loss setzen | OMS (Rules-basiert) | Ausschliesslich ATR-basiert. LLM hat keinen Einfluss auf numerische Stop-Levels |
| Taktische Parameter steuern | LLM (bounded Enums) | trail_mode, exit_bias, profit_protection_profile -- aber nur innerhalb deterministic Clamps |
| Hard-Stop (Tagesverlust) | Risk Gate | Ultimative Sicherheit, nicht verhandelbar |
| Order-Execution | OMS | Rein technisch, deterministisch |

> **Eiserne Regel:** Das LLM hat niemals direkten Zugriff auf Broker-APIs. Kein LLM-Output DARF direkt einen Trigger ausloesen. LLM-Features fliessen in die Rules Engine, die eigenstaendig und deterministisch entscheidet.

### 5.4 Weitere Prinzipien

| Prinzip | Bedeutung |
|---------|----------|
| **Simulationsfaehigkeit als First-Class-Concern** | Live und Simulation durchlaufen denselben Codepfad. Unterschiede nur ueber austauschbare Adapter und die Clock |
| **Determinismus** | Gleicher Input fuehrt zum gleichen Ergebnis. Keine Zufallskomponente |
| **MarketClock** | Einzige Zeitquelle im Trading-Codepfad. Kein `Instant.now()` |
| **Lean** | Professionell aber nicht over-engineered. Erst bauen, dann messen, dann optimieren |
| **Events sind immutable** | Alle Events sind unveraenderlich nach Erzeugung |

### 5.5 Warum ein LLM und nicht nur Quant?

Klassische quantitative Algorithmen arbeiten mit festen mathematischen Regeln. Sie sind exzellent in stationaeren Marktphasen, aber kaempfen mit:

- **Regime-Uebergaengen:** Der Moment, in dem ein Aufwaertstrend kippt, liegt per Definition ausserhalb der bisherigen statistischen Verteilung
- **Kontextabhaengiger Volatilitaetsabnahme:** Die Intraday-Volatilitaet haengt von Nachrichtenlage, Sektor, Earnings-Kalender und Marktphase ab
- **Muster-Erkennung in verrauschten Daten:** Ein LLM kann Muster (Doppelboden, Erschoepfungsluecken, Volumenanomalien) holistisch bewerten

Das LLM bringt die adaptive Intelligenz fuer Situationsbeurteilung. Die Deterministic Rules Engine bringt die reproduzierbare Entscheidungslogik. Die Quant-Schicht bringt die mathematische Disziplin fuer Risikokontrolle.

> **Stakeholder-Prinzip:** Aktie im Plus, Algo im Minus = Anomalie/Fehler -- immer untersuchen. Optimierungen muessen allgemeingueltig sein, nicht auf einzelne Aktien oder Chartverlaeufe optimiert (Overfitting vermeiden).

---

## 6. Vier-Schichten-Architektur

### 6.1 Schichtenmodell

1. **Datenschicht (Data Layer)**
   - Marktdaten empfangen, validieren (DQ-Gates), puffern (Rolling Buffers), Resampling (1m -> 3m/10m), Event-Emission
   - VWAP ist Source-of-Truth ausschliesslich in der Datenschicht

2. **Entscheidungsschicht (Brain Layer)**
   - QuantEngine (KPIs + Pattern-Features), LLM Engine (Analyst + Tactical Parameter Controller), Strategy Rules (FSM), Decision Arbiter, Regime Fusion
   - Hier findet die gesamte Entscheidungslogik statt

3. **Risiko- und Ausfuehrungsschicht (Risk & Execution Layer)**
   - Risk Gate (Position Sizing, Limits, R/R), OMS (Stops/Targets/Scaling/Trailing), Broker-Adapter
   - Safety hat Prioritaet ueber alle anderen Schichten

4. **Praesentation und Monitoring (Control/Observability)**
   - SSE Gateway (Monitoring-Streams), REST Controls (Kill-Switch, Pause/Resume), React Dashboard
   - Audit Logs, Metrics, Replay/Simulation Runner, Operator Controls

### 6.2 Fuenf zentrale Ports

Diese Ports sind die Mindest-Schnittstellen, damit Live und Simulation denselben Codepfad nutzen:

| Port | Verantwortung | Live-Implementierung | Simulation-Implementierung |
|------|--------------|---------------------|---------------------------|
| **MarketClock** | Zeit, Kalender, Session-Phasen | SystemMarketClock (Exchange-TZ `America/New_York`) | SimClock (vom Runner gesteuert) |
| **MarketDataFeed** | Liefert 1m Bars | IbMarketDataFeed (TWS API) | HistoricalMarketDataFeed (Replay) |
| **BrokerGateway** | Orders/Fills/Cancels | IbBrokerGateway (TWS API) | BrokerSimulator (Fill-Modell) |
| **LlmAnalyst** | Liefert JSON-Analyse | ClaudeAnalystClient / OpenAiAnalystClient | CachedAnalyst (aufgezeichnete Responses) |
| **EventLog** | Append-only persistieren | PostgresEventLog (async) | PostgresEventLog (synchron in Barrier) |

### 6.3 Backend-Module (10 Maven-Module)

| Modul | Zweck |
|-------|-------|
| `odin-api` | Ports, DTOs, Enums, Events, RunContext |
| `odin-broker` | IB-Adapter (Live) + BrokerSimulator (Sim) |
| `odin-data` | Data Pipeline, Buffer, DQ Gates, HistoricalFeed (Sim) |
| `odin-brain` | LLM-Clients, KPI, Rules, Quant, Arbiter |
| `odin-execution` | Risk Gate, OMS |
| `odin-core` | Pipeline-Orchestrierung, SimulationRunner, Clock, Global Risk, Lifecycle |
| `odin-audit` | EventRecord, PostgresEventLog, AuditEventDrainer |
| `odin-persistence` | DataSource-Config, JPA-Infrastruktur, Flyway (keine fachlichen Inhalte) |
| `odin-frontend` | React App (separater Build) |
| `odin-app` | Spring Boot Main, Mode-Wiring (Live/Sim), Konfiguration |

**Base-Package:** `de.its.odin`

**Abhaengigkeitsregeln (strikt):**

- `odin-api` hat keine Abhaengigkeiten (nur JDK + Validation-API)
- Fachmodule (broker, data, brain, execution, audit) haengen nur von `odin-api` ab -- nicht voneinander
- `odin-core` haengt von allen Fachmodulen ab -- es orchestriert die Pipeline
- `odin-app` haengt von `odin-core` ab (transitiv alle Module)
- **Verboten:** Fachmodul -> Fachmodul, Fachmodul -> odin-core, Zyklen

**Instanziierungsmodell:**

- **Singletons:** IbSession, IbDispatcher, MarketClock, EventLog, GlobalRiskManager, DataSource, alle Repositories
- **Pro Pipeline:** DataPipelineService, KpiEngine, RulesEngine, DecisionArbiter, RiskGate, OMS, PipelineStateMachine
- Pro-Pipeline-Services werden von `PipelineFactory` manuell instanziiert (POJOs, keine Spring-Beans)

**DDD-Modulschnitt Persistenz:**
Entities leben in ihren fachlichen Domaenen-Modulen (brain, execution, core, audit), nicht zentral. `odin-persistence` = reine Infrastruktur (DataSource, JPA, Flyway). Zwei Schreibpfade: direkte Domain-Persistierung + AuditEventDrainer nur fuer EventRecord.

---

## 7. Regime Detection -- Drei Ansaetze, empirisch zu loesen

Die Regime-Bestimmung ist ein zentrales Designproblem, fuer das drei Ansaetze existieren. Die endgueltige Loesung wird **empirisch** ueber das Evaluationsdesign ermittelt (Ablation A-E + FusionLab F0-F3).

### Ansatz 1: LLM-primaer (Fachkonzept v1.5)

Das LLM bestimmt das Regime allein. Die Quant-Schicht validiert nur.

### Ansatz 2: KPI-primaer, LLM-sekundaer (Master-Konzept)

Deterministische KPI-Regeln bestimmen das Regime. Bei Widerspruch zwischen KPI und LLM gilt das konservativere Regime (Konservativitaets-Rangfolge im Long-Only-Kontext: UNCERTAIN > TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > TREND_UP).

### Ansatz 3: Reliability-weighted Fusion (Build-Spec)

Dynamische Gewichtung basierend auf historischer Kalibrierung. Reliability wird auf Decision-Ebene aktualisiert (50-150 Datenpunkte pro Tag statt nur 1x/Tag). Korrelation wird ueber EMA/Downweighting geloest. Regime-Sparsity wird ueber hierarchisches Pooling mit Shrinkage geloest (globaler Score + Regime-Adjustments).

Die vollstaendige Spezifikation der Regime-Detection, FusionLab-Varianten (F0-F3) und der empirischen Kalibrierung findet sich in **02-regime-detection.md**.

---

## 8. Timeframe-Entscheidung (Stakeholder-Entscheidung)

| Timeframe | Rolle | Begruendung |
|-----------|-------|-------------|
| **1-Minute** | Rohsignal + Monitoring | Schnelle Ereigniserkennung (Crash, Spike, Exhaustion Climax, Strukturbruch) |
| **3-Minute** | Quant-Decision-Frame | Weniger Noise als 1m, bessere Stabilitaet fuer Entry/Exit-Entscheidungen |
| **10-Minute** | Bestaetigung/Hysterese | Trend-Quercheck, Fakeout-Filter, Regime-Persistenz |

Zielkonflikt (Noise vs. Latenz) wird durch **Hierarchie** geloest statt durch "eine perfekte Bargroesse". Die 3m-Ebene ist der primaere Decision-Takt; 1m liefert Events, 10m liefert Bestaetigung.

---

## 9. Glossar

| Begriff | Bedeutung |
|---------|----------|
| ADR14 | Average Daily Range ueber 14 Tage |
| ATR | Average True Range -- Mass fuer Volatilitaet, berechnet ueber N Perioden |
| Cycle | Vollstaendiger Entry-bis-Exit Run auf demselben Instrument am selben Tag |
| Decision-Bar | 3-Minuten-Bar als primaerer Decision-Trigger |
| EOD-Flat | End-of-Day Flat -- alle Positionen am Tagesende geschlossen |
| FSM | Finite State Machine -- Zustandsmaschine |
| GTC | Good-Til-Cancel -- Order bleibt aktiv bis explizit storniert |
| Hard-Stop | Absolutes Verlustlimit (-10% Tageskapital) |
| Highwater-Mark | Stop kann nur steigen, nie fallen |
| Kill-Switch | Notfall-Mechanismus: Sofort alle Positionen schliessen |
| MFE | Maximum Favorable Excursion -- hoechster unrealisierter Gewinn |
| OCA | One-Cancels-All -- Broker-seitige Order-Gruppe (NICHT verwendet fuer Stop/Target) |
| Quant-Only-Modus | Degraded Mode ohne LLM: Keine neuen Trades, bestehende Positionen trailen |
| R | Risk-Multiple: 1R = Distanz Entry zu Initial Stop |
| RTH | Regular Trading Hours (US: 09:30--16:00 ET) |
| Runner | Kleiner Restbestand (10--20%) mit Trailing-Stop fuer Trend-Days |
| Scaling-Out | Stufenweiser Positionsabbau in R-basierten Tranchen |
| TACTICAL_EXIT | LLM-gesteuerter Exit (bounded, KPI-bestaetigt) |
| TRAIL_ONLY | OMS-Override: alle Targets stornieren, nur Trail |
| VWAP | Volume Weighted Average Price -- Source-of-Truth in odin-data |
