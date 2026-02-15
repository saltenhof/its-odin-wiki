# ODIN — Kapitel 0: Systemuebersicht & Architekturkontext

**ODIN** = Orderflow Detection & Inference Node

Version: 0.7 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R1+R2+R3. Cross-Review R1+R2. DDD-Modulschnitt: 10 Module, odin-audit neu, odin-persistence reduziert

---

## 1. Zweck dieses Dokuments

Dieses Kapitel definiert die Systemgrenzen, Technologie-Entscheidungen, Leitprinzipien und die grobe Architekturstruktur von ODIN. Es bildet das Dach fuer die nachfolgenden Detailkapitel.

**Fachkonzept-Referenz:** `intraday-agent-concept.md` v1.5

---

## 2. Systemkontext

### Was ist der Agent?

ODIN ist ein vollautomatischer Intraday-Trading-Agent fuer Aktien (US, Europa, weitere Maerkte moeglich). Der Agent kombiniert:

- **LLM-Situationsanalyse** (Regime-Erkennung, Kontextsignale) — das LLM liefert strukturierte Features und Unsicherheiten, nie Handlungsanweisungen
- **Deterministische Regellogik** (Entry/Exit-Entscheidungen) — Rules Engine und Decision Arbiter treffen alle Handelsentscheidungen
- **Quantitative Validierung** (Indikatoren, Veto-Mechanismus)

Der Agent operiert als **Multi-Instrument (2–3 parallel), Single-Day, Long-Only** System mit vollstaendigem EOD-Flat-Constraint.

### Multi-Instrument-Architektur

Die Software unterstuetzt den parallelen Handel von 2–3 Instrumenten. Jedes Instrument wird durch eine **vollstaendig isolierte Pipeline** (eigene Instanz von Data Pipeline, Brain, Execution) verarbeitet. Es gibt keine fachliche Kopplung zwischen den Pipelines — jede Pipeline hat ihren eigenen Zustand, Buffer, KPIs und Entscheidungslogik.

**Geteilte Ressourcen** (infrastrukturell, nicht fachlich):
- IB-Gateway-Verbindung (eine Socket-Verbindung, multiplexed ueber Request-IDs) — oder SimulatedBroker in Simulation
- PostgreSQL-Datenbank (getrennte Datensaetze, gemeinsames Schema)
- LLM-API-Zugang (geteilter API-Key, separate Calls pro Pipeline) — oder CachedAnalyst in Simulation
- React-Frontend (Dashboard zeigt alle Pipelines)
- **MarketClock** (LiveClock oder SimClock)

**Isolierte Ressourcen** (pro Pipeline):
- Data Buffer, DQ Gates
- KPI-Berechnungen (ta4j-Instanzen)
- LLM-Kontext und -Aufrufe
- Rules Engine State, Pattern-State-Machines
- Quant Validation, Decision Arbiter
- Risk Gate (eigenes Tagesbudget pro Pipeline)
- OMS (eigene Orders, eigene Positionen)
- Agent-Zustandsmaschine

> **Globale Guardrails:** Ein uebergeordneter Risk-Manager ueberwacht die Summe aller Pipelines. Aggregiertes Tages-Exposure und aggregierter Tagesverlust werden gegen globale Limits geprueft. Bei Limit-Breach: alle offenen Orders werden cancelled, bestehende Positionen werden zum Market geschlossen (Flatten), der Agent stoppt den Handel fuer den Rest des Tages. Der globale Kill-Switch stoppt alle Pipelines gleichzeitig.

### Systemgrenzen

```
                        ┌───────────────────────────────────────────────────────┐
                        │                        ODIN                           │
                        │                                                       │
  Instrument-Vorgabe ──>│  ┌─ Pipeline 1 ──────────────────────────────┐       │
  (2-3 pro Tag)         │  │ Data Pipeline → Brain → Execution         │       │
                        │  └───────────────────────────────────────────┘       │
                        │  ┌─ Pipeline 2 ──────────────────────────────┐       │
                        │  │ Data Pipeline → Brain → Execution         │       │
                        │  └───────────────────────────────────────────┘       │
                        │                                                       │
                        │  ┌─ Global ──────────────────────────────────┐       │
                        │  │ Risk Manager │ Kill-Switch │ Lifecycle    │       │
                        │  └───────────────────────────────────────────┘       │
                        └───────┬────────────────────────┬─────────────────────┘
                                │                        │
               ┌────────────────┼────────────────┐       │
               │                │                │       │
               v                v                v       v
        ┌────────────┐  ┌────────────┐  ┌─────────┐  ┌────────────┐
        │ MarketData │  │ LLM       │  │ Postgre │  │ Broker     │
        │ Feed       │  │ Analyst   │  │ SQL     │  │ Gateway    │
        │            │  │           │  │         │  │            │
        │ Live: IB   │  │ Live:     │  │         │  │ Live: IB   │
        │ Sim: Hist. │  │  Claude/  │  │         │  │ Sim: Broker│
        │  Replay    │  │  OpenAI   │  │         │  │  Simulator │
        │            │  │ Sim:      │  │         │  │            │
        └────────────┘  │  Cached   │  └─────────┘  └────────────┘
                        └────────────┘
```

### Externe Systeme

| System | Anbindung | Richtung | Zweck |
|--------|-----------|----------|-------|
| IB Gateway | TWS API 10.40 (Socket) | Bidirektional | Marktdaten-Subscriptions, Order-Management, Account-Daten |
| Claude API | Claude Agent SDK (HTTPS) | Request/Response | Primaere LLM-Analyse (Regime, Opportunity Zones, Kontextsignale) |
| OpenAI API | REST/HTTPS | Request/Response | Alternative LLM-Analyse (pay-per-use, A/B-Test) |
| PostgreSQL | JDBC | Bidirektional | Trade-Logs, Performance-Daten, Konfiguration, Event-Log |

### Nicht im Scope (v1)

- Short-Selling
- Options/Futures
- Multi-Broker-Support
- Cloud-Deployment
- Mobile App

---

## 3. Leitprinzipien

### 3.1 Simulationsfaehigkeit als First-Class-Concern

Simulation ist kein Testfeature, sondern ein **Betriebsmodus**. Der gesamte Codepfad fuer Live und Simulation ist identisch — Unterschiede nur ueber austauschbare Adapter und die Clock.

**Architektonische Konsequenzen:**

- **Gleicher Codepfad:** Live und Simulation durchlaufen dieselbe Pipeline (Data → Brain → Execution). Kein separater Backtest-Engine-Code.
- **Port-Abstraktion:** Alle externen Abhaengigkeiten (Marktdaten, Broker, LLM, Uhrzeit) sind hinter Interfaces gekapselt. Je Betriebsmodus wird die passende Implementierung injiziert.
- **Marktzeit statt Systemzeit:** Alle Entscheidungen und Zeitstempel basieren auf der **MarketClock**, nicht auf `Instant.now()`. Im Live-Betrieb entspricht die MarketClock der Systemzeit; in der Simulation wird sie vom Runner kontrolliert.
- **Determinismus:** Jeder Simulationslauf ist reproduzierbar (gleiche Inputs → gleiche Outputs). Randomness nur ueber seeded RNG.
- **Record/Replay:** Der Live-Betrieb loggt alle entscheidungsrelevanten Events (Marktdaten, Broker-Events, LLM-Responses) so, dass sie 1:1 replaybar sind.

### 3.2 Vier zentrale Ports (Dual-Mode-Interfaces)

Die folgenden vier Schnittstellen sind die architektonischen Grenzen zwischen ODIN-Kernlogik und Aussenwelt. Jeder Port hat eine Live- und eine Simulation-Implementierung. Die Ports sind **bewusst minimal** gehalten — keine IB-spezifischen Methoden, keine Provider-spezifischen Features.

| Port | Live-Implementierung | Simulation-Implementierung |
|------|---------------------|---------------------------|
| **MarketClock** | SystemClock (Exchange-TZ) | SimClock (vom Runner gesteuert) |
| **MarketDataFeed** | IB Market Data Adapter (TWS API) | HistoricalMarketDataFeed (Replay aus DB/Dateien) |
| **BrokerGateway** | IB Order Adapter (TWS API) | BrokerSimulator (Fill-Modell, Slippage, Latenz) |
| **LlmAnalyst** | Provider-Client (Claude/OpenAI) | CachedAnalyst (aufgezeichnete Responses) oder Live-LLM mit Runner-Sync |

**Port-Contracts (Minimalanforderungen):**

**MarketClock:**
- `now()` liefert monotone Marktzeit in der Exchange-Zeitzone
- Kein direkter `Instant.now()` oder `System.currentTimeMillis()` im Trading-Codepfad — auch nicht im Logging. Einzige Zeitquelle fuer Entscheidungs- und Eventlogik ist die MarketClock
- Beinhaltet ein Session-Modell: Handelstage, Feiertage, Early Close, DST-Uebergaenge. In der Simulation entfaellt der Kalender (der Runner bestimmt den Tag)
- **Zeitzonen-Festlegung (v1):** Pro Run wird genau **ein Exchange-Kalender** konfiguriert (z.B. NYSE → `America/New_York`). Keine Mischmärkte innerhalb eines Runs. MarketEvents tragen `marketTime` in Exchange-TZ. Interne Verarbeitung in Exchange-TZ; UI konvertiert bei Bedarf in lokale Zeit. DST wird ueber `java.time.ZoneId` robust behandelt

**MarketDataFeed:**
- Liefert einen Stream von `MarketEvent`-Objekten, jedes mit `marketTime`, `instrumentId`, `sequenceNumber`
- Event-Ordering: Pro Instrument **total geordnet** (Sequence Number). Voraussetzung fuer deterministisches Replay
- Der Feed liefert Roh-Events (Ticks, L2-Updates). Bars sind ein **Derived Artifact** der Data Pipeline (BarBuilder). In der Simulation kann ein Feed auch direkt Bars liefern — mit dem expliziten Trade-off, dass Tick/L2-abhaengige Logik dann nicht validiert wird

**BrokerGateway:**
- **Asynchrones Event-Modell:** `submitOrder(intent) → orderId`, `cancelOrder(orderId)`. Ergebnisse kommen als asynchrone `BrokerEvent`-Callbacks (Accepted, Rejected, PartiallyFilled, Filled, Cancelled)
- **Idempotenz:** Jeder OrderIntent traegt eine `clientOrderId` als Idempotenz-Key. Retry und Replay erzeugen keine Doppel-Orders
- Positions-/Exposure-Snapshot fuer Recovery und Start-of-Day-Abgleich
- In der Simulation: BrokerSimulator liefert dieselben asynchronen Events (nicht synchron „sofort Filled")

**LlmAnalyst:**
- Output-Typ ist **kein Decision-Typ**. Keine Felder wie `action`, `buy`, `sell`, `size`
- Output enthaelt ausschliesslich: Features (`regime`, `confidence`, `opportunity_zones`, `risk_factors`), optionales Rationale (nur Logging), `marketTime` des Requests
- Arbiter/Rules konsumieren LLM-Output nur als Input fuer Scoring/Veto/Regime — nie als direkte Handlungsanweisung

### 3.3 SimulationRunner

Der SimulationRunner orchestriert einen simulierten Handelstag. Er nutzt dieselbe Pipeline wie der Live-Betrieb, speist aber historische Daten in kontrolliertem Tempo ein:

**Kernverantwortung:**
1. Laedt einen Handelstag (historische Bars/Ticks aus DB oder Datei)
2. Setzt die MarketClock auf Tagesstart (z.B. 07:00 ET fuer Pre-Market)
3. Streamt Marktdaten **inkrementell** — Event fuer Event, nicht alles auf einmal
4. Triggert denselben Decision-Mechanismus wie im Live-Betrieb
5. Simuliert Broker-Verhalten (Fills, Rejections, Partial Fills)
6. Erzwingt EOD-Flat und erzeugt einen Run-Report

**Geschwindigkeitskontrolle:**
- **Decision-synced Pace (Default):** Der Runner gibt Events frei bis ein Decision-Trigger kommt (z.B. Bar-Close). Dann greift die **Bar-Close-Barrier** (siehe unten). Nach Abschluss des Decision-Cycle wird die MarketClock weitergetaktet.
- **Echtzeit-Pace:** Events werden im originalen Zeitabstand freigegeben (interaktives Debugging/Demo).
- **Step-Modus:** Manuelles Weiterschalten (Event-fuer-Event oder Bar-fuer-Bar).
- **Steuerungsoperationen:** `play(speedFactor)`, `pause()`, `step(toNextBarClose | nBars | toTime)`.

**Bar-Close-Barrier (Determinismus-Garantie):**

Im Decision-synced-Modus muss der Runner sicherstellen, dass jeder Decision-Cycle vollstaendig abgeschlossen ist, bevor das naechste Event eingespeist wird. Die Barrier wartet auf Quiescence:
1. KPI-Compute abgeschlossen
2. Decision Loop (Rules + Quant + Arbiter) abgeschlossen
3. OMS/BrokerSimulator-Events bis zum definierten Cutoff verarbeitet
4. Globales Risk-Update abgeschlossen

Erst dann gibt der Runner das naechste Event frei und taktet die MarketClock weiter. Das verhindert Race Conditions und macht jeden Run deterministisch reproduzierbar.

**Multi-Pipeline-Koordination:** Die MarketClock ist global pro Run. Jede Pipeline hat ihren eigenen Decision-Trigger (Bar-Close pro Instrument). Der Runner koordiniert die Bar-Close-Barriers pro Pipeline und fuehrt zwischen den Barriers ein globales Risk-Update durch. Zusaetzlich reagiert Global Risk **sofort** auf Broker-Fill-Events (z.B. Stop-Loss-Trigger), auch ausserhalb der Barriers.

**Run-Identitaet:** Jeder Handelstag (Live) oder Simulationslauf erhaelt eine eindeutige `runId`. Diese dient als Join-Key fuer EventLog, ConfigSnapshot, Trades und Reports. Damit sind alle Artefakte eines Runs konsistent zuordenbar.

**LLM in der Simulation — zwei Profile:**

| Profil | LLM-Quelle | Geschwindigkeit | Einsatz |
|--------|-----------|-----------------|---------|
| **Cached (Default)** | Aufgezeichnete Responses (Key: Hash aus Snapshot + Prompt-Version + Model-ID) | Maximal (keine Wartezeit) | Massen-Backtests, Regressionstests, Parameteroptimierung |
| **Live-LLM** | Echte API-Calls | Begrenzt durch LLM-Inference | Qualitative Analyse, neue Szenarien, Prompt-Evaluation |

**LLM-Timing-Modell:** Jeder LLM-Request wird bei MarketTime `T` gestellt. Die Response gilt als verfuegbar bei MarketTime `T + Δt_model`. `Δt_model` ist konfigurierbar (Default: typische Provider-Latenz, z.B. 3s). Im Cached-Modus kann `Δt_model` auf 0 gesetzt werden (Massen-Backtest) oder auf einen realistischen Wert (Timing-Effekte testen). Im Live-LLM-Modus bestimmt die tatsaechliche Antwortzeit den Wert; die Simulation blockiert bis zur Response.

> **TTL/Freshness-Regel in Simulation:** Die TTL bezieht sich auf die **MarketClock**, nicht auf die Systemzeit. Eine LLM-Analyse ist „abgelaufen" wenn N Sekunden *Marktzeit* vergangen sind, unabhaengig davon wie schnell die Simulation laeuft.

### 3.4 LLM ist Analyst, nicht Policy-Maker

Das LLM liefert ausschliesslich **strukturierte Features und Unsicherheiten** in einem validierten JSON-Schema. Die Entscheidungslogik liegt vollstaendig in der deterministischen Rules Engine und dem Decision Arbiter.

**Technische Durchsetzung:**
- Strikte JSON-Schema-Validierung jeder LLM-Response
- **Whitelist** definiert, welche Felder als Features in die Entscheidungslogik einfliessen duerfen (z.B. `regime`, `confidence`, `opportunity_zones`)
- Freitext-Felder (`reasoning`, `notes`) fliessen ausschliesslich ins Logging
- Rules Engine + Arbiter treffen Entscheidungen basierend auf LLM-Features + KPI-Werten — nie auf Basis von LLM-„Empfehlungen"

---

## 4. Technologie-Stack

### Runtime & Frameworks

| Komponente | Technologie | Version | Begruendung |
|------------|-------------|---------|-------------|
| Sprache | Java | 21 (LTS) | Vorgabe. Type-Safety, Performance, Ecosystem |
| Framework | Spring Boot | 3.x | Vorgabe. DI, Konfiguration, Lifecycle-Management |
| Build | Maven | Multi-Modul | Bewaehrte Struktur fuer modulare Java-Projekte |
| Chart-KPIs | ta4j | aktuell | Vorgabe. Bewaehrte Bibliothek fuer technische Analyse |
| Datenbank | PostgreSQL | 17 | Vorgabe. Bestehende Infrastruktur |
| Frontend | React | 18+ | Vorgabe. Echtzeit-Dashboard fuer Trade-Monitoring |
| Broker-API | TWS API | 10.40 | Vorgabe. Neueste Version, Socket-basiert |

### LLM-Anbindung

| Provider | SDK/Protokoll | Einsatz |
|----------|--------------|---------|
| Anthropic (Claude) | Claude Agent SDK (Java) | Primaer. Strukturierte Analyse via Tool-Use/Structured-Output |
| OpenAI | OpenAI API (REST) | Alternativ. Pay-per-use fuer A/B-Tests oder Evaluation |

> **LLM-Abstraktion:** Eine gemeinsame Schnittstelle (`LlmAnalyst`-Port) kapselt beide Provider. Der aktive Provider wird **vor Handelsstart per Konfiguration** gewaehlt und steht fuer den gesamten Handelstag fest. Es gibt kein Runtime-Switching zwischen Providern. Faellt der aktive Provider aus, greift der Circuit Breaker und der Agent wechselt in den **Quant-Only-Modus** (kein Provider-Failover).

### Betriebsumgebung

| Aspekt | Spezifikation |
|--------|---------------|
| OS | Windows 11 |
| Deployment | Lokaler Prozess (kein Container, kein Cloud) |
| IB-Anbindung | IB Gateway (headless) laeuft als separater Prozess auf demselben Rechner |
| Datenbank | PostgreSQL laeuft lokal auf demselben Rechner |
| Netzwerk | Localhost fuer DB und IB Gateway. Ausgehend: LLM-APIs (HTTPS) |

---

## 5. Architektur-Ueberblick (Vier Schichten)

Die Architektur folgt dem im Fachkonzept definierten Vier-Schichten-Modell. Die technische Umsetzung bildet jede fachliche Schicht auf ein oder mehrere Spring-Boot-Module ab. **Alle Schichten arbeiten gegen die Port-Interfaces (Abschnitt 3.2), nicht gegen konkrete Implementierungen.**

### Schicht 1: Datenschicht (Data Pipeline)

**Verantwortung:** Marktdaten empfangen, validieren, puffern und an nachgelagerte Schichten verteilen.

**Kernkomponenten:**
- **MarketDataFeed (Port):** Abstrahiert die Datenquelle. Live: IB Market Data Adapter (reqRealTimeBars, reqMktData, reqMktDepth, reqHistoricalData). Simulation: HistoricalMarketDataFeed (Replay).
- **Data Quality Gates:** Stale-Detection, Outlier-Filter, Crash-Detection (VOR Outlier), Bar-Completeness.
- **Rolling Data Buffer:** In-Memory-Ringpuffer fuer die letzten N Bars + voraggregierte Werte. Single Source of Truth fuer alle Berechnungen innerhalb eines Handelstages.

> **Snapshot-Prinzip:** Bei jedem Decision-Cycle erzeugt der Buffer einen **immutable MarketSnapshot** (Kurs, Volumen, L2, Indikatoren, MarketClock-Timestamp). Alle nachgelagerten Berechnungen (KPI, Rules, Quant, Arbiter) arbeiten auf demselben Snapshot — keine parallelen Reads auf mutierenden Daten. Determinismus garantiert.

### Schicht 2: Entscheidungsschicht (Brain)

**Verantwortung:** Marktdaten interpretieren und Handelsentscheidungen treffen.

**Kernkomponenten:**
- **LlmAnalyst (Port):** Baut Kontext-Payload auf, ruft LLM-API (oder Cache), parsed und validiert die strukturierte Antwort. Event-driven.
- **KPI Engine (ta4j):** Berechnet technische Indikatoren (VWAP, RSI, ATR, EMA, Bollinger, ADX, Volumen-Profil). Parallelisierbar ueber mehrere Timeframes.
- **Rules Engine:** Implementiert die deterministischen Entry/Exit-Regeln pro Regime. Pattern-State-Machines.
- **Quant Validation:** Berechnet quant_score, prueft Veto-Bedingungen.
- **Decision Arbiter:** Kombiniert Rules-Intent + Quant-Score zu finaler Entscheidung. Reicht Trade-Intent an Risk Gate weiter.

### Schicht 3: Risiko- und Ausfuehrungsschicht (Execution)

**Verantwortung:** Position Sizing, Risikopruefung, Order-Erzeugung und -Ueberwachung.

**Kernkomponenten:**
- **Risk Gate:** Position Sizing (Fixed Fractional Risk), Pre-Trade-Limit-Pruefung, Tagesbudget-Tracking, FX-Conversion.
- **Order Management System (OMS):** Erzeugt und verwaltet Orders. Multi-Tranchen-Management. Fill-Event-Handling. Stop-Nachfuehrung.
- **BrokerGateway (Port):** Abstrahiert die Orderausfuehrung. Live: IB Order Adapter (TWS API). Simulation: BrokerSimulator (Fill-Modell, konfigurierbare Slippage und Latenz).

> **Broker-State als Source of Truth:** Im Live-Betrieb ist der IB-Account-State autoritativ. In der Simulation uebernimmt der BrokerSimulator diese Rolle. Das OMS synchronisiert sich in beiden Modi gegen den BrokerGateway.

### Schicht 4: Praesentation & Monitoring

**Verantwortung:** Echtzeit-Visualisierung, Alerts, manueller Kill-Switch.

**Kernkomponenten:**
- **SSE Gateway + REST Controls:** Streamt Agent-State, Trade-Updates, P&L, KPI-Werte an das React-Frontend (SSE). Controls (Kill-Switch, Pause/Resume) ueber REST POST (Kap 9).
- **React Dashboard:** Echtzeit-Anzeige von Chart, Agent-State, Orders/Positionen, P&L, LLM-Analyse-Historie, Alerts.
- **Alert Engine:** Eskalationsstufen (INFO → WARN → ALERT → CRITICAL).
- **Manual Controls:** Kill-Switch, Pause/Resume, Parameter-Override.

---

## 6. Parallelisierungs-Strategie

### Event-Driven Core

Der Agent arbeitet **event-driven**, nicht polling-basiert. Marktdaten-Events treiben die Pipeline:

```
MarketDataFeed Event (Tick/Bar)
  ├── async → Rolling Buffer Update
  ├── async → KPI Recalculation (parallel ueber Timeframes)
  ├── async → Data Quality Check
  └── (bei Decision-Trigger) → Decision Loop
```

> **Decision-Trigger:** Der primaere Trigger fuer den Decision Loop ist der **Bar-Close** (z.B. 1-Minute-Bar). Ticks aktualisieren den Buffer und Trailing-Stops, loesen aber keinen vollstaendigen Decision-Cycle aus. Details in Kapitel 2 und 4.

### Thread-Modell (Ueberblick)

| Bereich | Modell | Scope |
|---------|--------|-------|
| IB-Receiver + Dispatcher | Single-threaded Empfang, Dispatch auf Pipeline-Queues | Global |
| Data Pipeline | Asynchrone Buffer-Updates, DQ-Checks | Pro Pipeline |
| KPI-Compute | Parallel pro Timeframe/Indikator (CPU-bound) | Pro Pipeline |
| Decision Loop | Strikt sequentiell pro Decision-Cycle | Pro Pipeline |
| LLM-IO | Asynchron, Single-Flight (max. 1 ausstehender Call) | Pro Pipeline |
| Order-Execution | Strikt sequentiell (Konsistenz) | Pro Pipeline |
| Frontend-Push | Asynchrones Event-Streaming | Global |
| Global-Risk | Aggregiertes Exposure-Monitoring | Global |

> **Kritischer Pfad (pro Pipeline, ohne LLM):** MarketDataFeed-Event → Buffer → KPI → Decision → Risk → BrokerGateway. Der LLM-Call ist asynchron und blockiert den kritischen Pfad nicht. Pipelines laufen vollstaendig parallel — keine Synchronisation untereinander.
>
> **In der Simulation:** Der SimulationRunner steuert das Tempo. Im Decision-synced-Modus gibt er das naechste Event erst frei, wenn der aktuelle Decision-Cycle abgeschlossen ist — Threading-Modell bleibt identisch.

### Backpressure

- KPI langsamer als Event-Rate: Events gepuffert, aelteste verworfen (Ring-Buffer-Semantik)
- LLM laeuft: Neue Anfragen gequeued (Single-Flight, max. 1 ausstehend)
- IB nicht erreichbar: Heartbeat → Reconnect (bounded retries) → Kill-Switch

---

## 7. Modulgliederung (Maven)

Modulschnitt (10 Maven-Module). Wird in Kapitel 1 detailliert.

```
odin/
├── odin-api/          # Shared Interfaces (Ports!), DTOs, Enums
├── odin-broker/       # IB Gateway Adapter (TWS API 10.40), BrokerSimulator
├── odin-data/         # Data Pipeline, Buffer, DQ Gates, HistoricalFeed (pro Pipeline)
├── odin-brain/        # LLM Service, KPI Engine, Rules, Quant, Arbiter + LlmCallEntity, DecisionLogEntity
├── odin-execution/    # Risk Gate, OMS + TradingRunEntity, TradeEntity, FillEntity, ConfigSnapshotEntity
├── odin-core/         # Pipeline-Orchestrierung, SimulationRunner, Global Risk, Lifecycle, Clock + PipelineStateEntity, DailyPerformanceEntity
├── odin-audit/        # EventRecordEntity, PostgresEventLog, AuditEventDrainer (flaches Event-Archiv)
├── odin-persistence/  # DataSource-Config, JPA-Infrastruktur, Schema-Migration (keine fachlichen Inhalte)
├── odin-frontend/     # React App (separater Build)
├── odin-app/          # Spring Boot Main, Konfiguration, Modus-Auswahl (Live/Sim)
└── pom.xml            # Parent POM
```

> **DDD-Modulschnitt:** Entities und Repositories leben in ihren fachlichen Domaenen-Modulen. odin-persistence ist auf reine Infrastruktur reduziert. odin-audit ist neu fuer das flache Event-Archiv.
> **Abhaengigkeitsrichtung:** api ← broker, data, brain, execution, audit ← core ← app. odin-persistence ist transitive Infrastruktur fuer persistierende Module. Kein Zyklus.
> **Instanziierung:** Die Module data, brain, execution werden pro Pipeline-Instanz erzeugt (Services als POJOs). Repositories sind Singletons. Der broker-Adapter und core sind Singletons.
> **Port-Interfaces:** Die fuenf zentralen Ports (MarketClock, MarketDataFeed, BrokerGateway, LlmAnalyst, EventLog) leben in odin-api. Implementierungen in den jeweiligen Modulen.

---

## 8. Datenfluss (End-to-End)

```
┌──────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│ MarketData   │────>│ Data Pipeline│────>│    Brain     │────>│  Execution   │
│ Feed (Port)  │     │              │     │              │     │              │
│              │     │ - DQ Gates   │     │ - KPI Engine │     │ - Risk Gate  │
│ Live: IB     │     │ - Buffer     │     │ - LLM (Port) │     │ - OMS        │
│ Sim: Hist.   │     │ - Snapshot   │     │ - Rules      │     │ - Broker     │
│              │     │              │     │ - Quant      │     │   (Port)     │
│              │     │              │     │ - Arbiter    │     │              │
└──────────────┘     └──────────────┘     └─────────────┘     └──────┬───────┘
                                                                      │
                           ┌──────────────┐                           │
                           │  PostgreSQL   │<──────────────────────────┘
                           │              │     (Event-Log, Trades)
                           │ - Events     │
                           │ - Trades     │
                           │ - P&L        │
                           │ - LLM Logs   │
                           │ - Config     │
                           └──────────────┘

                           ┌──────────────┐
                           │   React UI   │
                           │              │
                           │ - Chart      │
                           │ - State      │
                           │ - P&L        │
                           │ - Controls   │
                           └──────────────┘
```

---

## 9. Konfigurationsarchitektur

Folgt den bewaehrten Prinzipien der CSpec (R13). Wichtige Namespaces:

```
odin.agent.*                    # Agent-Lifecycle, Zustandsmaschine, Pipeline-Anzahl
odin.agent.mode=LIVE|SIMULATION # Betriebsmodus
odin.agent.pipelines.*          # Pipeline-spezifische Konfiguration (pro Instrument)
odin.broker.*                   # IB-Verbindung, Gateway-Host/Port
odin.data.*                     # Buffer-Groessen, DQ-Schwellenwerte
odin.kpi.*                      # ta4j-Parameter, Timeframes
odin.llm.*                      # Provider, Model, Temperature, Timeout
odin.rules.*                    # Entry/Exit-Schwellenwerte, Regime-Mapping
odin.risk.*                     # Sizing, Limits, Hard-Stop (global + pro Pipeline)
odin.execution.*                # Order-Typen, Repricing, Tranchen
odin.simulation.*               # SimulationRunner: Datenquelle, Pace-Modus, LLM-Profil
odin.simulation.broker.*        # BrokerSimulator: Slippage-Modell, Fill-Latenz
odin.frontend.*                 # SSE-Intervalle, Auth, Chart-Defaults
odin.persistence.*              # Retention, Batch-Groessen
```

> **Konfigurations-Snapshot:** Zu Beginn jedes Handelstages (Live) oder Simulationslaufs wird ein vollstaendiger Konfigurations-Snapshot persistiert:
> - Alle aktiven Properties (Key-Value)
> - Git-SHA / Build-Artifact-Version
> - Prompt-Version (LLM-Schema)
> - LLM-Model-ID und Provider
>
> Damit sind Simulationsergebnisse reproduzierbar und vergleichbar. Zusammen mit dem Event-Log (inkl. Sequence Numbers und LLM-Cache-Keys) entsteht ein vollstaendiger Reproduzierbarkeitsnachweis.

---

## 10. Querschnittsthemen

### Logging & Event-Log

- **Structured Logging** (JSON) fuer maschinelle Auswertung
- **Event-Log** (append-only): Alle entscheidungsrelevanten Events (MarketData, Broker-Events, LLM-Calls, Decisions, State-Transitions) werden persistiert — fuer Forensik, Recovery und Replay
- Jeder LLM-Call, jeder Trade-Intent, jeder Fill wird protokolliert
- **Retention:** Konfigurierbar. Default: 180 Tage fuer Roh-Events, Aggregates/Trades dauerhaft

### Error Handling & Resilience

- **Circuit Breaker** fuer LLM-API-Calls (3 Failures → Quant-Only-Modus)
- **Heartbeat** fuer IB Gateway (Ausfall → Kill-Switch)
- **Graceful Degradation (Quant-Only-Modus):** Wenn der LLM-Provider ausfaellt, verwaltet der Agent bestehende Positionen weiter (Stop-Nachfuehrung, EOD-Flat), eroeffnet aber **keine neuen Positionen**
- **IB Reconnect/Resync:** Nach Verbindungsverlust automatische Subscription-Rehydration. Offene Positions und Orders werden gegen den Broker-State abgeglichen (Broker ist Source of Truth). Order-Submissions sind idempotent (Client-Order-ID).
- **Crash-Recovery (v1-Entscheidung):** ODIN ist in v1 **nicht intraday-restartable fuer Trading**. Bei Prozesscrash waehrend des Handelstages: bestehende Stop-Orders bleiben beim Broker aktiv (GTC-Stops schuetzen offene Positionen). WinSW startet den Prozess automatisch neu (Kap 10 §5), aber die **Startup-Recovery erkennt den Crash** (TradingRun mit `outcome=ERROR` fuer den aktuellen Handelstag, oder TradingRun ohne `endTime`) und erzwingt den **Safe-Mode**: Trading ist fuer den Rest des Tages gesperrt, nur Dashboard/Reconciliation/Forensik sind verfuegbar. Der Operator kann den Broker-State im Dashboard pruefen. Das EventLog ermoeglicht Forensik und Replay des abgebrochenen Tages. Beim naechsten regulaeren Tagesstart: Reconciliation der Broker-Positionen vor Handelsbeginn (Flat-Check)

### Security

- Broker-Credentials und API-Keys ausschliesslich via Environment-Variablen
- Kein Credentials in Code, Config oder Logs
- LLM-Prompts enthalten keine Credentials oder Account-Informationen

### Observability

Messbarkeit ist Voraussetzung fuer den Live-Betrieb. Folgende Metriken werden ueber Micrometer/JMX exponiert:

| Metrik | Zielwert | Messung |
|--------|----------|---------|
| E2E-Latenz (Event → Order, ohne LLM) | < 100ms (p99) | Histogram pro Pipeline |
| LLM Response Time | < 10s (p95) | Timer pro Call |
| DQ Drop Rate | < 1% pro Session | Counter pro Pipeline |
| Queue Depth (Dispatch) | < 50 Events | Gauge |
| Order Roundtrip (Submit → Ack) | IB-abhaengig, gemessen | Timer |

> **Hinweis:** Die Zielwerte sind initiale Annahmen und werden nach den ersten Live-Sessions gegen Messdaten validiert. Der Order-Roundtrip haengt primaer von IB ab und wird gemessen, nicht als SLO vorgegeben.

### Operationelle Anforderungen (v1)

Die folgenden Punkte werden in **Kapitel 10 (Deployment & Betrieb)** detailliert:

- **Event-/State-Store:** Jede Pipeline persistiert ihren Zustand fuer Recovery und Replay
- **Windows Service + Watchdog:** Agent laeuft als Windows Service. WinSW startet den Prozess bei Absturz automatisch neu; bei Crash waehrend RTH erzwingt die Startup-Recovery den Safe-Mode (kein Trading, Kap 10 §5)
- **Kill-Switch ohne UI:** REST-Endpoint und CLI-Command als Fallback
- **Log Rotation & Retention:** Konfigurierbare Rotation, kein Disk-Overflow

### Simulation & Validierung

Die Simulation ist kein separates Subsystem, sondern derselbe Agent im Simulationsmodus (siehe Abschnitt 3). Der Validierungsprozess folgt dem Fachkonzept (Kap. 17):

| Stufe | Modus | Beschreibung |
|-------|-------|--------------|
| Unit/Integration | Mock/Replay | Isolierte Komponententests, Pipeline-Tests mit aufgezeichneten Daten |
| Day-Simulation | SimulationRunner | Einzelne Handelstage mit historischen Intraday-Daten durchspielen |
| Paper-Trading | Live gegen IB Paper Account | Vollstaendiger Agent, echte Marktdaten, simulierte Orders |
| Klein-Live | Live gegen echten Account | 10% Kapital, reale Bedingungen |

> **Simulation Fidelity (v1-Annahmen):** Das Fill-Modell des BrokerSimulators wird in Kapitel 3 definiert. Minimale Annahmen:
> - **Decision** erfolgt auf Bar-Close `t`. **Fill** erfolgt auf Bar-Open `t+1` + konfigurierbare Slippage. Das verhindert unrealistische „Fill in same Bar"-Effekte.
> - Keine Partial Fills in v1.
> - Pending Orders leben ueber Bars hinweg — OMS und Risk Gate muessen damit umgehen.
> - Kostensimulation gemaess Fachkonzept (IB-Tiered-Pricing, Spread, Slippage).

---

## 11. Offene Architekturentscheidungen

| # | Frage | Vorlaeufige Empfehlung | Begruendung | Status |
|---|-------|------------------------|-------------|--------|
| A1 | Eigenstaendige Anwendung oder MOSES-Submodul? | **(a) Eigenstaendiges Projekt** | Komplett unabhaengig von MOSES | ENTSCHIEDEN |
| A2 | Frontend-Kommunikation? | **SSE (Monitoring) + REST POST (Controls)** | SSE fuer unidirektionales Streaming; REST POST fuer Kill-Switch/Pause (Kap 9) | ENTSCHIEDEN |
| A3 | ta4j direkt oder Wrapper? | **(b) Leichtgewichtiger Wrapper** | Testbarkeit und einheitliche API | ENTSCHIEDEN |
| A4 | LLM-Response-Format? | **Strict JSON-Schema + Validierung** | Provider-agnostisch. Versionsfeld im Schema | VORLAEUFIG |
| A5 | State-Machine-Framework? | **(b) Eigene schlanke FSM** | Zustaende ueberschaubar, deterministisch | ENTSCHIEDEN |
| A6 | Echtzeit-Chart im Frontend? | **TradingView Lightweight Charts + Recharts** | Spezialisierte Libs | ENTSCHIEDEN |
| A7 | SimulationRunner-Modul? | **In odin-core** | Runner orchestriert Pipelines wie der Live-Lifecycle | VORLAEUFIG |
| A8 | Historical Data Format? | **PostgreSQL (Event-Log) + optional CSV** | DB fuer strukturierte Abfragen, CSV als Import-Format | VORLAEUFIG |
