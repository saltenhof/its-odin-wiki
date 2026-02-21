# ODIN -- Konsolidiertes Zielbild-Konzept

## 00 -- Systemuebersicht, Scope, Constraints, Designphilosophie

**Version:** Konsolidiert (Stand: 2026-02-21)
**Konsolidiert aus:** Fachkonzept v1.5, Strategie-Optimierung-Sparring, Master-Konzept v1.1, Unified Concept v2.1, Build-Spec v3.0, Evaluationsdesign, Testplan, Stakeholder-Feedback 1+2, L2-Kompromiss

> **Quellen:** Fachkonzept v1.5 (Kap. 1--3, 12), Master-Konzept v1.1 (Systemparameter, Designphilosophie, Module), Build-Spec v3.0 (Kap. 1--4), Unified Concept v2.1 (Schichten, Ports), Strategie-Sparring (L2-Kompromiss, OHLCV-Constraint), Stakeholder-Feedback 1+2 (Timeframes, Constraints)

> **Normative Sprache:** MUSS = zwingend erforderlich, DARF NICHT / NIE = verboten, SOLL = dringend empfohlen (Abweichung nur mit dokumentierter Begruendung), KANN = optional.

---

## 1. Was ist ODIN?

**ODIN** (Orderflow Detection & Inference Node) ist ein vollautomatischer Intraday-Trading-Agent fuer Aktien. V1 fokussiert ausschliesslich auf US-Maerkte (NYSE, NASDAQ); EU-Maerkte sind als spaetere Erweiterung vorgesehen. Das System kombiniert drei Saeulen:

1. **LLM-Situationsanalyse** -- Regime-Erkennung, Kontextsignale, taktische Parametersteuerung
2. **Deterministische Regellogik** -- Entry/Exit-Entscheidungen, Pattern-State-Machines, Quant-Scoring
3. **Quantitative Validierung** -- Indikatoren, Scoring-Modell, Veto-Mechanismus

Das System betreibt 1--3 isolierte Pipelines parallel. Jede Pipeline handelt genau ein Instrument pro Tag. Die Instrumente werden extern vorgegeben und beim Start der Pre-Market-Phase (07:00 ET) eingefroren -- kein Symbolwechsel waehrend des Handelstages. Am Ende des Handelstages MUESSEN alle Positionen aller Pipelines geschlossen sein (EOD-Flat).

---

## 2. Systemparameter und Rahmenbedingungen

| Parameter | Wert | Anmerkung |
|-----------|------|-----------|
| Tageskapital | 10.000 EUR | Aufgeteilt auf aktive Pipelines |
| Max. Tagesverlust (Hard-Stop) | -10% (1.000 EUR) | Global ueber alle Pipelines und alle Cycles |
| Instrumente | 1--3 parallel | Je Pipeline genau ein Instrument |
| Richtung | Long only | Kein Short |
| Instrumenttyp | Aktien | Keine Derivate |
| Datenbasis | 1-Min OHLCV + Volume (Minimum) | Konfigurierbar via `odin.data.subscriptions` (OHLCV_ONLY / BASIC_QUOTES / L2_DEPTH) |
| Rohsignal | 1m | Monitoring-Ebene |
| Quant-Decision-Frame | 3m | Resampled, weniger Noise |
| Confirmation-Frame | 10m | Trend-/Regime-Hysterese, Fakeout-Filter |
| Handelszeiten | Pre-Market + RTH | Exchange-konfigurierbar |
| Autonomie | Vollautomatisch | Mit manuellen Controls (Pause/Kill) |
| EOD-Flat | MUSS | Keine Overnight-Position |
| Max. Cycles pro Instrument/Tag | 3 (konfigurierbar) | `maxCyclesPerInstrumentPerDay` -- Hard-Cap als Guardrail |
| Max. Trades/Tag (global) | 5 | Global ueber alle Pipelines und Cycles |

**Instrumentauswahl (bewusste Designentscheidung):**
Instrument-Selektion ist extern durch den Operator. Instrumente werden handverlesen uebergeben. Ein automatischer Screening-, Validierungs- oder Universe-Ingestion-Filter ist nicht vorgesehen. Die Auswahl wird zu Session-Start eingefroren. Wird ein Instrument vor RTH gehalted oder mit einer Trading-Restriction belegt, wechselt die betroffene Pipeline in DAY_STOPPED (kein Ersatzsymbol, andere Pipelines laufen weiter).

**Anforderungen an geeignete Instrumente:**

- Aktien (V1: ausschliesslich US-Maerkte -- NYSE, NASDAQ; EU-Maerkte als geplante Erweiterung)
- Ausreichende Liquiditaet (Spread < 0.5%, taegl. Volumen > 500.000)
- Keine Penny Stocks (Preis > 5 USD)
- Handelbar ueber IB TWS

---

## 3. Scope und Nicht-Ziele

### In Scope

- Intraday Long-Trades auf Aktien
- Ein- und Ausstieg, Teilverkaeufe (Tranchen), Trailing-Stops, Re-Entry/Scale-In
- Multi-Cycle-Handling (mehrere Trades pro Tag pro Instrument, siehe Abschnitt 9)
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

### 4.1 Konfigurationsabhaengige Datenverfuegbarkeit

Die Datenverfuegbarkeit ist NICHT statisch, sondern konfigurationsabhaengig. ODIN MUSS in jeder Stufe funktionsfaehig sein. Die Stufe wird ueber `odin.data.subscriptions` konfiguriert:

| Stufe | Inhalt | Typischer Einsatz |
|-------|--------|-------------------|
| **OHLCV_ONLY** | Nur OHLCV-Bars | Backtest (Standard), Live-Minimalbetrieb |
| **BASIC_QUOTES** | OHLCV + Bid/Ask | Live mit echter Spread-Pruefung, bessere Execution |
| **L2_DEPTH** | OHLCV + Bid/Ask + Level-2-Orderbuch | Beste Execution-Qualitaet im Live-Betrieb |

**Zentrale Designregel:** Features, die optional sind, haben immer einen Proxy-Fallback. Alpha-Signale DUERFEN nur auf Daten basieren, die auch im Backtest desselben Laufs verfuegbar waren.

**Harter Satz:** Bid/Ask und L2-Daten DUERFEN live fuer Execution und Risk genutzt werden, aber NIE als Alpha-Feature -- es sei denn, der Backtest-Run basiert auf derselben Datenstufe (z.B. ueber aufgezeichnete Daten).

**Konsequenzen:**

- Alpha-Generierung, Regime-Erkennung und Setup-Aktivierung basieren im Standard ausschliesslich auf OHLCV
- Liquiditaets- und Ausfuehrungsqualitaet wird auf der Stufe OHLCV_ONLY ueber Proxies modelliert (Volumen/Range/Fill-Rate)
- Alle Signale MUESSEN aus OHLCV ableitbar sein, sofern kein Recording-basierter Backtest vorliegt
- Backtest-Runs werden mit ihrer Datenstufe getaggt, um Vergleichbarkeit sicherzustellen

**Recording-Modus:** Wenn Bid/Ask oder L2 im Live-Betrieb verfuegbar sind, KOENNEN diese Daten aufgezeichnet werden, um spaetere Backtests auf derselben Datenstufe zu ermoeglichen. Ein Recording-basierter Backtest auf BASIC_QUOTES oder L2_DEPTH DARF dann auch Features nutzen, die auf diesen Daten basieren.

Die vollstaendige Data/Feature-Matrix findet sich in **01-data-pipeline.md**, Abschnitt "Datenverfuegbarkeits-Matrix".

### 4.2 V1: Ausschliesslich US-Maerkte

V1 fokussiert ausschliesslich auf US-Maerkte (NYSE, NASDAQ). Handelszeiten, Kalender, Tick-Sizes und alle marktspezifischen Konfigurationen werden nur fuer den US-Markt spezifiziert. Begruendung: Die Kostenstruktur (Kommissionen, Spreads, Markttiefe) macht US-Intraday-Trading fuer Privatpersonen deutlich attraktiver als europaeische Maerkte.

EU-Maerkte (z.B. XETRA, Euronext) sind als spaetere Erweiterung vorgesehen, aber NICHT Teil der V1-Spezifikation. Die Architektur (Exchange-konfigurierbare Zeitzonen, MarketClock) ist auf Multi-Market-Betrieb vorbereitet, aber V1 implementiert und testet ausschliesslich gegen US-Boersen.

### 4.3 Zeitrestriktion

- Alle Zeiten beziehen sich auf die Boersen-Zeitzone des Instruments (`America/New_York` fuer US)
- **MarketClock** ist die einzige Zeitquelle im Trading-Codepfad. Kein `Instant.now()` in Decision-/Risk-/OMS-Logik
- Trading-Kalender MUSS Feiertage, halbe Handelstage, DST-Umstellungen korrekt abbilden

### 4.4 Namespace

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

## 9. Normative Begriffshierarchie

Die folgenden Begriffe sind systemweit verbindlich und MUESSEN in Code, Konfiguration, Logging und Dokumentation einheitlich verwendet werden.

### Trade

Ein **Trade** umfasst alles von der ersten Eroeffnung bis zur vollstaendigen Schliessung einer Position in einem Instrument. Aufstocken (Scale-In), Teilverkaeufe (Scale-Out), Trailing-Stop-Nachfuehrung -- alles ist Teil desselben Trades. Ein Trade endet erst, wenn die gesamte Position Null ist.

### Cycle

Ein **Cycle** ist ein erneuter Trade im selben Instrument am selben Tag. Voraussetzung: Die Position aus dem vorherigen Trade wurde vollstaendig geschlossen. Erst dann beginnt ein neuer Cycle. Cycle 1 = erster Trade, Cycle 2 = Re-Entry nach vollstaendigem Exit, usw.

Limit: `maxCyclesPerInstrumentPerDay` (Default: 3, konfigurierbar). Dieses Limit ist ein Hard-Cap als Guardrail gegen Overtrading.

### Tranche

Eine **Tranche** ist ein Teilstueck innerhalb eines Trades. Jede Aufstockung (Scale-In) eroeffnet eine neue Tranche. Jeder Teilverkauf (Scale-Out) schliesst eine Tranche. Die initiale Position ist Tranche 1.

Limit: `maxTranchesPerTrade` (konfigurierbar). Begrenzt die Komplexitaet des Positionsmanagements innerhalb eines einzelnen Trades.

### Abgrenzung

| Begriff | Granularitaet | Beispiel |
|---------|--------------|---------|
| **Trade** | Gesamte Position von Eroeffnung bis Flat | Entry 100 Shares -> Scale-In 50 -> Trail -> Exit alles = 1 Trade |
| **Cycle** | Erneuter Trade im selben Instrument/Tag | Trade 1 komplett geschlossen -> neuer Entry = Cycle 2 |
| **Tranche** | Teilstueck innerhalb eines Trades | Initial 100 Shares (T1) + Scale-In 50 Shares (T2) = 2 Tranchen |

---

## 10. Glossar

| Begriff | Bedeutung |
|---------|----------|
| ADR14 | Average Daily Range ueber 14 Tage |
| ATR | Average True Range -- Mass fuer Volatilitaet, berechnet ueber N Perioden |
| Cycle | Erneuter Trade im selben Instrument am selben Tag. Position komplett geschlossen -> neu eroeffnet = neuer Cycle (siehe Abschnitt 9) |
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
| Scaling-Out | Stufenweiser Positionsabbau durch Schliessen einzelner Tranchen (siehe Abschnitt 9) |
| TACTICAL_EXIT | LLM-gesteuerter Exit (bounded, KPI-bestaetigt) |
| Trade | Gesamte Position von Eroeffnung bis vollstaendige Schliessung in einem Instrument (siehe Abschnitt 9) |
| TRAIL_ONLY | OMS-Override: alle Targets stornieren, nur Trail |
| Tranche | Teilstueck innerhalb eines Trades -- jede Aufstockung/Teilverkauf ist eine Tranche (siehe Abschnitt 9) |
| VWAP | Volume Weighted Average Price -- Source-of-Truth in odin-data |
