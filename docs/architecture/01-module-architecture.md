# ODIN — Kapitel 1: Modularchitektur & Package-Struktur

Version: 0.5 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R1+R2+R3. DDD-Modulschnitt: odin-persistence aufgeloest, Entities in Domaenen-Module, odin-audit neu

---

## 1. Zweck dieses Kapitels

Dieses Kapitel detailliert den in Kapitel 0 definierten Modulschnitt: Verantwortlichkeiten, Abhaengigkeiten, Package-Struktur und Instanziierungsmodell jedes Maven-Moduls. Es ist die Referenz fuer die Abhaengigkeitsrichtung, die Platzierung der Port-Implementierungen (Live/Sim) und den Scope jedes Moduls.

---

## 2. Moduluebersicht

ODIN besteht aus 10 Maven-Modulen plus Parent POM:

```
odin/                               (Parent POM)
├── odin-api/                       Ports, DTOs, Enums, Events, RunContext
├── odin-broker/                    IB-Adapter (Live) + BrokerSimulator (Sim)
├── odin-data/                      Data Pipeline, Buffer, DQ Gates, HistoricalFeed (Sim)
├── odin-brain/                     LLM-Clients, KPI, Rules, Quant, Arbiter + LlmCallEntity, DecisionLogEntity
├── odin-execution/                 Risk Gate, OMS + TradingRunEntity, TradeEntity, FillEntity, ConfigSnapshotEntity
├── odin-core/                      Pipeline-Orchestrierung, SimulationRunner, Clock, Global Risk, Lifecycle + PipelineStateEntity, DailyPerformanceEntity
├── odin-audit/                     EventRecordEntity, PostgresEventLog, AuditEventDrainer
├── odin-persistence/               DataSource-Config, JPA-Infrastruktur, Schema-Migration (keine fachlichen Inhalte)
├── odin-frontend/                  React App (separater Build-Prozess)
└── odin-app/                       Spring Boot Main, Mode-Wiring (Live/Sim), Konfiguration, @EntityScan, @EnableJpaRepositories
```

> **DDD-Modulschnitt:** Entities und Repositories leben in ihren fachlichen Domaenen-Modulen (brain, execution, core, audit), nicht in einem zentralen Persistence-Modul. `odin-persistence` ist auf reine Infrastruktur reduziert (DataSource, JPA-Setup, Flyway). Jedes Domaenen-Modul persistiert seine strukturierten Daten direkt ueber eigene Repositories. Nur das flache Event-Archiv (`EventRecord`) wird ueber den `AuditEventDrainer` in `odin-audit` geschrieben.

---

## 3. Abhaengigkeitsgraph

```
                            ┌───────────┐
                            │  odin-api │  (keine Abhaengigkeiten)
                            └─────┬─────┘
                                  │
       ┌───────────┬──────────────┼──────────────┬──────────┬──────────┐
       │           │              │              │          │          │
 ┌─────┴─────┐ ┌───┴───┐ ┌───────┴──────┐ ┌────┴─────┐ ┌─┴────┐    │
 │odin-broker│ │odin-  │ │odin-         │ │odin-     │ │odin- │    │
 │           │ │data   │ │execution     │ │audit     │ │brain │    │
 └─────┬─────┘ └───┬───┘ └───────┬──────┘ └────┬─────┘ └──┬───┘    │
       │           │              │              │          │        │
       └───────────┴──────────────┴──────────────┴──────────┘        │
                                  │                                   │
                           ┌──────┴──────┐                            │
                           │  odin-core  │────────────────────────────┘
                           │ (Orchestr.) │
                           └──────┬──────┘
                                  │
                           ┌──────┴──────┐
                           │  odin-app   │
                           │ (Main, DI)  │
                           └─────────────┘

    odin-persistence (Infrastruktur):
      → Transitive Dependency fuer brain, execution, core, audit
      → Liefert: DataSource-Config, JPA, PostgreSQL-Driver
      → KEINE fachlichen Inhalte
```

### Abhaengigkeitsregeln (strikt)

**Erlaubt:**
- **odin-api** hat keine Abhaengigkeiten (ausser JDK + Validation-API)
- **Fachmodule** (broker, data, brain, execution, audit) haengen nur von **odin-api** ab — nicht voneinander
- **Persistierende Fachmodule** (brain, execution, core, audit) haengen zusaetzlich von **odin-persistence** ab (transitive Infrastruktur: DataSource, JPA, PostgreSQL-Driver)
- **odin-core** haengt von allen Fachmodulen ab — es orchestriert die Pipeline
- **odin-app** haengt von odin-core ab (transitiv alle Module)
- **odin-persistence** ist ein reines Infrastruktur-Modul — es hat **keine fachlichen Abhaengigkeiten** (nur odin-api fuer Shared Types)

**Verboten:**
- Fachmodule duerfen **nicht** voneinander abhaengen (kein brain→broker, kein execution→data, kein audit→brain)
- Fachmodule duerfen **nicht** von odin-core oder odin-app abhaengen
- BrokerSimulator (in odin-broker) darf **nicht** von OMS/Execution abhaengen — nur von odin-api
- **Keine Zyklen.** Brain greift nie direkt auf Broker zu — nur ueber die von odin-core gesteuerte Data-Pipeline
- Kein Modul ausser odin-app haengt von Spring Boot Starter Web ab (kein HTTP/UI-Kram in Fachmodulen)
- **odin-frontend** ist ein separater Build (npm/React), keine Maven-Abhaengigkeit zu Java-Modulen
- **odin-persistence** darf keine Fachmodule importieren — es kennt keine Entities anderer Module

> **Enforcement:** Dependency-Regeln werden automatisiert geprueft: ArchUnit-Tests (Package/Layer/Forbidden-Dependencies) und Maven Enforcer Plugin (verbotene transitive Dependencies in falschen Modulen). Ohne automatisierte Pruefung erodieren die Regeln schleichend.

---

## 4. Package-Konvention

**Base-Package:** `de.its.odin`

Jedes Modul hat sein eigenes Sub-Package mit einheitlicher interner Struktur:

| Modul | Package | Beispiel |
|-------|---------|---------|
| odin-api | `de.its.odin.api` | `de.its.odin.api.port.MarketDataFeed` |
| odin-broker | `de.its.odin.broker` | `de.its.odin.broker.ib.IbBrokerGateway` |
| odin-data | `de.its.odin.data` | `de.its.odin.data.buffer.RollingDataBuffer` |
| odin-brain | `de.its.odin.brain` | `de.its.odin.brain.rules.RulesEngine` |
| odin-execution | `de.its.odin.execution` | `de.its.odin.execution.oms.OrderManagementService` |
| odin-core | `de.its.odin.core` | `de.its.odin.core.pipeline.TradingPipeline` |
| odin-audit | `de.its.odin.audit` | `de.its.odin.audit.service.PostgresEventLog` |
| odin-persistence | `de.its.odin.persistence` | `de.its.odin.persistence.config.DataSourceConfiguration` |
| odin-app | `de.its.odin.app` | `de.its.odin.app.OdinApplication` |

### Interne Sub-Packages je Modul

```
de.its.odin.{modul}/
├── config/         # @Configuration, @ConfigurationProperties (nur wo noetig)
├── model/          # Modul-interne Datentypen (Records, Enums)
├── service/        # Geschaeftslogik
├── persistence/    # Entity + Repository (nur in persistierenden Domaenen-Modulen)
│   ├── entity/     # JPA Entities
│   └── repository/ # Spring Data JPA Repositories
├── ib/             # IB-spezifische Implementierung (nur in odin-broker)
├── live/           # Produktive Implementierung (nicht IB-spezifisch, z.B. LLM-Clients, SystemClock)
├── sim/            # Simulation-Implementierung
└── exception/      # Modul-spezifische Exceptions
```

> **Konvention:** `ib/` = IB-spezifischer Code (nur in odin-broker). `live/` = produktiver Code ohne IB-Bezug (z.B. LLM-Clients, SystemMarketClock). `sim/` = Simulation. `persistence/` = JPA Entities und Spring Data Repositories (nur in brain, execution, core). Keine weiteren Kategorien.

> **Trennung Live/Sim:** Module, die einen Port implementieren, trennen Live- und Sim-Implementierungen in separate Sub-Packages. Das verhindert versehentliche Kopplung zwischen Live- und Sim-Code.

> **Vertical Slices:** Jedes persistierende Domaenen-Modul besitzt seine Entities und Repositories in einem `persistence/` Sub-Package. Die Entities gehoeren zur Domaene, nicht zu einem zentralen Persistence-Modul. odin-persistence liefert nur die Infrastruktur (DataSource, JPA-Setup).

### odin-api Struktur (nur Contracts)

```
de.its.odin.api/
├── port/           # Die fuenf zentralen Ports (Interfaces)
│   ├── MarketClock
│   ├── MarketDataFeed
│   ├── BrokerGateway
│   ├── LlmAnalyst
│   └── EventLog
├── event/          # Event-Typen (Records): MarketEvent, BrokerEvent, LlmAnalysis, RunEvent
├── dto/            # Shared DTOs: MarketSnapshot, TradeIntent, QuantScore
├── model/          # Shared Enums: PipelineState, Regime, TradeDirection, OrderType, OrderStatus, RuntimeMode
└── context/        # RunContext
```

> **Kein Spring-Kontext** in odin-api. Keine Implementierungen, keine externen Abhaengigkeiten. Nur Typen und Interfaces.

---

## 5. Fuenf zentrale Ports (Alignment mit Kapitel 0)

Die in Kapitel 0 (Abschnitt 3.2) definierten Ports leben als Interfaces in `de.its.odin.api.port`. Jeder Port hat mindestens eine Live- und eine Sim-Implementierung:

| Port (odin-api) | Live-Implementierung | Modul | Sim-Implementierung | Modul |
|-----------------|---------------------|-------|---------------------|-------|
| `MarketClock` | `SystemMarketClock` | odin-core | `SimClock` | odin-core |
| `MarketDataFeed` | `IbMarketDataFeed` | odin-broker | `HistoricalMarketDataFeed` | odin-data |
| `BrokerGateway` | `IbBrokerGateway` | odin-broker | `BrokerSimulator` | odin-broker |
| `LlmAnalyst` | `ClaudeAnalystClient`, `OpenAiAnalystClient` | odin-brain | `CachedAnalyst` | odin-brain |
| `EventLog` | `PostgresEventLog` | odin-audit | `PostgresEventLog` (selbe Impl.) | odin-audit |

> **EventLog** wird in beiden Modi identisch implementiert (PostgreSQL) und lebt in **odin-audit**. In der Simulation werden dieselben Events geloggt wie im Live-Betrieb — das ist Voraussetzung fuer Record/Replay. Der `AuditEventDrainer` in odin-audit schreibt ausschliesslich `EventRecord` (flaches Archiv). Strukturierte Daten (Trades, Fills, LLM-Calls etc.) werden von den Domaenen-Modulen direkt ueber eigene Repositories persistiert.

### RunContext

`RunContext` ist ein immutable Record in `de.its.odin.api.context` mit folgenden Feldern:
- `runId` — eindeutige ID pro Handelstag/Simulationslauf
- `mode` — `RuntimeMode.LIVE` oder `RuntimeMode.SIMULATION`
- `exchange` — Exchange-Kalender (z.B. NYSE)
- `tradingDate` — Handelstag
- `promptVersion` — LLM-Schema-Version
- `modelId` — LLM-Model-ID
- `provider` — LLM-Provider (Claude/OpenAI)
- `configSnapshotId` — Verweis auf persistierten Konfigurations-Snapshot

`RunContext` wird in **odin-core** (LifecycleManager) erzeugt und an alle Pipeline-Komponenten uebergeben. Jedes persistierte Event traegt die `runId` als Join-Key.

---

## 6. Modul-Steckbriefe

### odin-api

**Verantwortung:** Gemeinsame Typen und Contracts. Definiert die Vertraege zwischen den Modulen.

**Inhalte:**
- **Ports (Interfaces):** `MarketClock`, `MarketDataFeed`, `BrokerGateway`, `LlmAnalyst`, `EventLog`
- **Events (Records):** `MarketEvent` (marketTime, instrumentId, sequenceNumber, payload), `BrokerEvent` (Accepted/Rejected/PartiallyFilled/Filled/Cancelled), `LlmAnalysis`, `RunEvent`
- **DTOs (Records):** `MarketSnapshot` (immutable), `TradeIntent`, `QuantScore`
- **Enums:** `PipelineState`, `Regime`, `TradeDirection`, `OrderType`, `OrderStatus`, `RuntimeMode`
- **Context:** `RunContext`
- **Kein Spring-Kontext**, keine Implementierungen, keine externe Abhaengigkeit

### odin-broker

**Verantwortung:** Broker-Kommunikation — sowohl Live (IB) als auch simuliert.

**Inhalte (Live — `de.its.odin.broker.ib`):**
- `IbSession` — **Singleton, owning the socket.** Verwaltet die eine TWS-API-Verbindung, Receiver-Thread, Heartbeat, Reconnect. Kein anderer Baustein darf eine zweite IB-Verbindung oeffnen
- `IbDispatcher` — Demultiplexed IB-Callbacks auf Pipeline-spezifische Queues (Request-ID-basiert)
- `IbMarketDataFeed` — implements `MarketDataFeed`. Pro Pipeline instanziiert als Facade ueber `IbSession`. Filtert/abonniert genau ein Instrument, vergibt Sequence Numbers
- `IbBrokerGateway` — implements `BrokerGateway`. Pro Pipeline instanziiert als Facade ueber `IbSession`. Namespace fuer clientOrderId (Pipeline-Prefix), filtert BrokerEvents auf eigenes Instrument
- `IbAccountAdapter` — Account-Daten, Positions-Abfrage fuer Reconciliation

**Inhalte (Simulation — `de.its.odin.broker.sim`):**
- `BrokerSimulator` — implements `BrokerGateway`. Fill-Modell (v1: Fill auf naechstem Bar-Open + konfigurierbare Slippage), asynchrone Event-Emission (nicht synchron "sofort Filled"), Latenz-Simulation

**Scope:** `IbSession` + `IbDispatcher` sind Singletons (eine IB-Verbindung). `IbMarketDataFeed` und `IbBrokerGateway` werden **pro Pipeline** als Facades ueber die Singleton-Session erzeugt. `BrokerSimulator` wird **pro Pipeline** instanziiert (damit Orders/Positions isoliert bleiben)

**Subscription-Ownership:** `IbSession` owned den tatsaechlichen IB-Subscription-Lifecycle (subscribe/unsubscribe auf TWS-API-Ebene). Die Facades requesten/registrieren Subscriptions, aber `IbSession` setzt um und fuehrt Referenzzaehlung. Das verhindert, dass eine Pipeline die Subscriptions einer anderen Pipeline mit abmeldet.

**Shutdown-Reihenfolge:** Pipeline stop → Facade deregistriert → IbSession unsubscribed (wenn Referenzzaehler 0) → IbDispatcher stop als Letztes

**Abhaengigkeiten:** odin-api, TWS API 10.40

### odin-data

**Verantwortung:** Marktdaten empfangen, validieren, puffern und als immutable Snapshots bereitstellen.

**Inhalte (`de.its.odin.data`):**
- `RollingDataBuffer` — Ringpuffer fuer Bars, Ticks, L2-Daten
- `BarBuilder` — Baut Bars aus Tick-Events (Derived Artifact)
- `MarketSnapshotFactory` — Erzeugt immutable `MarketSnapshot` bei jedem Decision-Cycle (inkl. MarketClock-Timestamp)
- `DataQualityGate` — Stale-Detection, Outlier-Filter, Crash-Detection, Bar-Completeness
- `DataPipelineService` — Orchestriert Feed-Event → Buffer-Update → DQ-Check → Snapshot-Erzeugung

**Inhalte (Simulation — `de.its.odin.data.sim`):**
- `HistoricalMarketDataFeed` — implements `MarketDataFeed`. Replay aus DB oder Dateien (CSV). Event-Ordering via Sequence Number. Kann Ticks oder direkt Bars liefern (expliziter Trade-off: bei reinem Bar-Replay werden Tick/L2-abhaengige Logikpfade nicht validiert)

**Scope:** Pro Pipeline instanziiert (jede Pipeline hat eigene Buffer-Instanzen)

**Abhaengigkeiten:** odin-api

### odin-brain

**Verantwortung:** Interpretation der Marktdaten und Erzeugung von Trade-Intents. Persistierung von LLM-Calls und Decision-Logs.

**Inhalte (Live — `de.its.odin.brain.live`):**
- `ClaudeAnalystClient` — implements `LlmAnalyst`. Claude SDK, structured output
- `OpenAiAnalystClient` — implements `LlmAnalyst`. OpenAI REST API, JSON-Mode

**Inhalte (Simulation — `de.its.odin.brain.sim`):**
- `CachedAnalyst` — implements `LlmAnalyst`. Cache-Key: Hash(Snapshot + Prompt-Version + Model-ID). Response-Lookup, optionale Δt_model-Verzoegerung

**Inhalte (Kern — `de.its.odin.brain`):**
- `KpiEngine` — ta4j-Wrapper, parallele Berechnung ueber Timeframes
- `RulesEngine` — Deterministische Entry/Exit-Regeln, Pattern-State-Machines
- `QuantValidation` — quant_score Berechnung, Veto-Logik
- `DecisionArbiter` — Kombiniert Rules-Intent + Quant-Score + LlmAnalysis-Features → `TradeIntent`

**Inhalte (Persistenz — `de.its.odin.brain.persistence`):**
- `LlmCallEntity` — LLM-Aufruf-Protokollierung + Cache-Source (Kap 8)
- `DecisionLogEntity` — Decision-Cycle-Protokollierung (Kap 8)
- `LlmCallRepository`, `DecisionLogRepository` — Spring Data JPA Repositories

**Scope:** Services pro Pipeline instanziiert (eigener LLM-Kontext, eigene KPIs, eigener Rules-State). Repositories als Singletons (Spring Beans).

**Abhaengigkeiten:** odin-api, odin-persistence (transitive JPA-Infra), ta4j, Claude SDK, OpenAI SDK (LLM-SDKs nur in den jeweiligen Adapter-Klassen)

### odin-execution

**Verantwortung:** Risikopruefung, Position Sizing, Order-Erzeugung und -Verwaltung. Persistierung von Trades, Fills, TradingRuns und ConfigSnapshots.

**Inhalte (Kern — `de.its.odin.execution`):**
- `RiskGate` — Position Sizing (Fixed Fractional Risk), Tagesbudget-Tracking, Pre-Trade-Limit-Pruefung, FX-Conversion
- `OrderManagementService` — Multi-Tranchen-Management, Stop-Nachfuehrung, Fill-Event-Handling. Arbeitet ausschliesslich gegen den `BrokerGateway`-Port. Persistiert Trades und Fills direkt ueber eigene Repositories
- `TranchenCalculator` — Berechnet Tranchenaufteilung (3/4/5 Tranchen budgetabhaengig)

**Inhalte (Persistenz — `de.its.odin.execution.persistence`):**
- `TradingRunEntity` — RunContext-Abbild: finalPnl, tradeCount, contractCurrency = Execution-Semantik (Kap 8)
- `TradeEntity` — Entry-bis-Exit Round-Trip, OMS-verwaltet (Kap 8)
- `FillEntity` — Broker-Ausfuehrung, OMS-verarbeitet (Kap 8)
- `ConfigSnapshotEntity` — Teil des TradingRun-Kontexts, Reproduzierbarkeit (Kap 8)
- `TradingRunRepository`, `TradeRepository`, `FillRepository`, `ConfigSnapshotRepository` — Spring Data JPA Repositories

**Scope:** Services pro Pipeline instanziiert (eigenes Tagesbudget, eigene Orders). Repositories als Singletons (Spring Beans).

**Abhaengigkeiten:** odin-api, odin-persistence (transitive JPA-Infra). Keine Abhaengigkeit zu odin-broker — nur Port-Interface

### odin-core

**Verantwortung:** Pipeline-Orchestrierung, Lifecycle-Management, Simulation, globale Koordination. Persistierung von Pipeline-State und Tages-Performance.

**Inhalte (Kern — `de.its.odin.core`):**
- `TradingPipeline` — Verbindet data → brain → execution fuer ein Instrument
- `PipelineFactory` — Erzeugt und konfiguriert Pipeline-Instanzen. Verdrahtet ausschliesslich gegen Port-Interfaces. Erhaelt alle benoetigten Repositories per Constructor Injection und reicht sie an Pipeline-Services weiter
- `PipelineStateMachine` — Eigene schlanke FSM (INITIALIZING → OBSERVING → ... → EOD)
- `LifecycleManager` — Tages-Lifecycle (Pre-Market → RTH → EOD), erzeugt RunContext + runId
- `GlobalRiskManager` — Aggregiertes Exposure-Monitoring ueber alle Pipelines, globale Limits. Reagiert zwischen Barriers (regulaer) und sofort auf Fill-Events (Kill-Switch)
- `KillSwitchService` — Globaler Kill-Switch (UI, REST-Endpoint, CLI)

**Inhalte (Clock — `de.its.odin.core.clock`):**
- `SystemMarketClock` — implements `MarketClock`. Live-Modus: Exchange-TZ, Session-Kalender
- `SimClock` — implements `MarketClock`. Simulation-Modus: vom Runner gesteuert

**Inhalte (Simulation — `de.its.odin.core.sim`):**
- `SimulationRunner` — Orchestriert simulierten Handelstag. Geschwindigkeitskontrolle (Decision-synced, Echtzeit, Step), Bar-Close-Barrier, Multi-Pipeline-Koordination. Keine Handelslogik — nur Clock treiben, Events einspeisen, Barriers kontrollieren, Reports schreiben

**Inhalte (Persistenz — `de.its.odin.core.persistence`):**
- `PipelineStateEntity` — State-Recovery beim Startup, Lifecycle-Domaene (Kap 8)
- `DailyPerformanceEntity` — Tages-Aggregat ueber alle Pipelines (Kap 8)
- `PipelineStateRepository`, `DailyPerformanceRepository` — Spring Data JPA Repositories

**Scope:** Singleton (eine Instanz verwaltet alle Pipelines)

**Abhaengigkeiten:** odin-api, odin-broker, odin-data, odin-brain, odin-execution, odin-audit, odin-persistence (transitive JPA-Infra)

### odin-audit

**Verantwortung:** Flaches Append-Only Event-Archiv (EventRecord) fuer Forensik und Replay. EventLog-Port-Implementierung.

**Inhalte (`de.its.odin.audit`):**
- `PostgresEventLog` — implements `EventLog` (Port aus odin-api). Append-only Event-Persistierung fuer Record/Replay
- `AuditEventDrainer` — Konsumiert den EventLog-Spool und schreibt **ausschliesslich EventRecords**. Kein Schreiben in strukturierte Tabellen — das machen die Domaenen-Module direkt

**Inhalte (Persistenz — `de.its.odin.audit`):**
- `EventRecordEntity` — Flaches Archiv aller Events (Kap 8)
- `EventRecordRepository` — Spring Data JPA Repository

**Inhalte (Konfiguration — `de.its.odin.audit.config`):**
- `AuditProperties` — Drain-Intervall, Batch-Groesse, Retention

**Scope:** Singleton (eine Instanz, Pipeline-ID + runId als Diskriminatoren in den EventRecords)

**Abhaengigkeiten:** odin-api, odin-persistence (transitive JPA-Infra)

### odin-persistence

**Verantwortung:** Reine Datenbank-Infrastruktur. DataSource-Konfiguration, JPA-Setup, Schema-Migration (Flyway). **Keine fachlichen Inhalte** — keine Entities, keine Repositories, keine Geschaeftslogik.

**Inhalte (`de.its.odin.persistence.config`):**
- `DataSourceConfiguration` — DataSource-Bean, Connection-Pool (HikariCP)
- JPA-Infrastruktur-Setup (EntityManagerFactory, TransactionManager)
- Flyway-Konfiguration (Schema-Migration)

**Scope:** Singleton (eine DataSource fuer die gesamte Anwendung)

**Abhaengigkeiten:** odin-api (fuer Shared Types), Spring Data JPA, PostgreSQL Driver, Flyway

> **Infrastruktur-Modul:** odin-persistence ist ein reines Infrastruktur-Modul. Es wird als transitive Dependency von den persistierenden Domaenen-Modulen (brain, execution, core, audit) genutzt, stellt aber selbst keine fachlichen Beans bereit.

### odin-app

**Verantwortung:** Spring Boot Hauptanwendung. Composition Root. Mode-Wiring. Konfiguration.

**Inhalte:**
- `OdinApplication` — `@SpringBootApplication` Main-Klasse
- **Mode-Wiring:**
  - `LiveWiringConfig` — `@Configuration`, aktiv bei `odin.agent.mode=LIVE`. Bindet: `SystemMarketClock`, `IbMarketDataFeed`, `IbBrokerGateway`, `ClaudeAnalystClient`/`OpenAiAnalystClient`
  - `SimWiringConfig` — `@Configuration`, aktiv bei `odin.agent.mode=SIMULATION`. Bindet: `SimClock`, `HistoricalMarketDataFeed`, `BrokerSimulator`, `CachedAnalyst`, `SimulationRunner`
- **SSE Controller + REST Controls** — Backend-Endpoints fuer Frontend-Kommunikation (Kap 9)
- **REST Controller** — Kill-Switch-Endpoint (`/api/kill`), Health, Admin-Endpoints
- Properties-Import-Kaskade (odin-app → odin-core → alle Fachmodule)
- `@EnableConfigurationProperties` fuer alle Properties-Klassen
- `@EnableJpaRepositories` und `@EntityScan` fuer alle persistierenden Domaenen-Module
- Actuator-Konfiguration (Health, Metrics)

**JPA-Konfiguration:**
```java
@EnableJpaRepositories(basePackages = {
    "de.its.odin.execution.persistence.repository",
    "de.its.odin.brain.persistence.repository",
    "de.its.odin.core.persistence.repository",
    "de.its.odin.audit.repository"
})
@EntityScan(basePackages = {
    "de.its.odin.execution.persistence.entity",
    "de.its.odin.brain.persistence.entity",
    "de.its.odin.core.persistence.entity",
    "de.its.odin.audit.entity"
})
```

**Scope:** Singleton (eine JVM, eine Spring-Boot-Instanz)

**Abhaengigkeiten:** odin-core (transitiv alle anderen Module)

> **Spring-Annotation-Policy:** `@Repository` wird in den persistierenden Domaenen-Modulen (brain, execution, core, audit) fuer Spring Data JPA Repositories verwendet — diese sind Singletons. `@Component` und `@Service` werden nur in Singleton-Modulen (odin-broker, odin-audit, odin-app) verwendet. Pro-Pipeline-**Services** (odin-data, odin-brain, odin-execution) werden **manuell** von der PipelineFactory instanziiert — sie sind keine Spring-Beans. Component-Scan ist auf `de.its.odin.core`, `de.its.odin.broker`, `de.its.odin.audit` und `de.its.odin.app` beschraenkt. Repositories werden ueber `@EnableJpaRepositories` separat aktiviert.

### odin-frontend

**Verantwortung:** React-Dashboard fuer Echtzeit-Monitoring und Controls.

**Build:** Separater npm-Build, kein Maven-Dependency-Graph mit Java-Modulen.

**Kommunikation mit Backend:** SSE (Monitoring-Streams) + REST POST (Kill-Switch, Pause/Resume, Kap 9)

---

## 7. Instanziierungsmodell

**Global Singleton (pro Run/Process):**

| Komponente | Modul | Begruendung |
|------------|-------|-------------|
| `RunContext`-Erzeugung (LifecycleManager) | odin-core | Ein Run, eine ID |
| `MarketClock` (System oder Sim) | odin-core | Eine Uhr pro Run |
| `EventLog` (PostgresEventLog) | odin-audit | Ein Writer pro Run |
| `AuditEventDrainer` | odin-audit | Drain EventLog → EventRecord |
| `IbSession` + `IbDispatcher` | odin-broker | Eine Socket-Verbindung |
| LLM Provider HTTP-Client (Rate Limits) | odin-brain | Ein Client, geteilte Rate Limits |
| PostgreSQL `DataSource` | odin-persistence | Eine DB-Verbindung |
| `GlobalRiskManager` | odin-core | Aggregiert ueber alle Pipelines |
| `SimulationRunner` (nur im Sim-Modus) | odin-core | Orchestriert den gesamten Run |
| Alle Repositories | brain, execution, core, audit | Spring Data JPA Singletons |

**Pro Pipeline (pro Instrument):**

| Komponente | Modul | Begruendung |
|------------|-------|-------------|
| `DataPipelineService`, `RollingDataBuffer`, `BarBuilder`, `DataQualityGate` | odin-data | Eigener Buffer + DQ-State |
| `KpiEngine`, `RulesEngine`, `QuantValidation`, `DecisionArbiter` | odin-brain | Eigene KPIs, eigener Rules-State |
| `RiskGate`, `OrderManagementService` | odin-execution | Eigenes Budget, eigene Orders |
| `IbMarketDataFeed` (Facade ueber IbSession) | odin-broker | Filtert auf ein Instrument |
| `IbBrokerGateway` (Facade ueber IbSession) | odin-broker | clientOrderId-Namespace pro Pipeline |
| `BrokerSimulator` (nur Sim-Modus) | odin-broker | Isolierte Orders/Positions pro Pipeline |
| `HistoricalMarketDataFeed` (nur Sim-Modus) | odin-data | Replay-Stream pro Instrument |
| LLM-Kontext/Cache-Keying (nicht der HTTP-Client) | odin-brain | Pipeline-spezifischer Kontext |
| `PipelineStateMachine` | odin-core | Eigene FSM pro Pipeline |

> **Pro-Pipeline-Instanzen werden manuell von der PipelineFactory erzeugt** (plain Java objects, kein Spring Prototype Scope). Component-Scan ist auf `de.its.odin.core`, `de.its.odin.broker`, `de.its.odin.audit` und `de.its.odin.app` beschraenkt. Repositories sind Singletons (via `@EnableJpaRepositories`) und werden von der PipelineFactory per Constructor Injection an die Pro-Pipeline-POJOs weitergereicht.

### Pipeline-Erzeugung

Die `PipelineFactory` in odin-core erzeugt fuer jedes konfigurierte Instrument eine vollstaendige Pipeline. Die Verdrahtung erfolgt **ausschliesslich gegen Port-Interfaces aus odin-api**:

```
PipelineFactory.createPipeline(instrument, runContext, ports, repositories)
  → new DataPipelineService(buffer, dqGate, snapshotFactory, clock)   // clock: MarketClock
  → new KpiEngine(ta4jConfig)
  → new RulesEngine(rulesConfig)
  → new QuantValidation(quantConfig)
  → new DecisionArbiter(rules, quant, llmAnalyst, decisionLogRepo)   // llmAnalyst: LlmAnalyst Port
  → new RiskGate(riskConfig, pipelineBudget)
  → new OrderManagementService(brokerGateway, eventLog,               // brokerGateway: BrokerGateway Port
                               tradeRepo, fillRepo)                   // Repositories fuer direkte Persistierung
  → new TradingPipeline(data, brain, execution, fsm, runContext)
```

> **Contract-Only Wiring + Repository Injection:** Die PipelineFactory erhaelt die Port-Implementierungen (MarketClock, MarketDataFeed, BrokerGateway, LlmAnalyst, EventLog) und die Singleton-Repositories per Constructor Injection von odin-app. Sie instanziiert die Pipeline-internen Komponenten manuell und reicht die Repositories an die Services weiter. Die Services persistieren ihre strukturierten Daten direkt (OMS → Trade/Fill, DecisionArbiter → DecisionLog). Der EventLog-Port wird weiterhin fuer das flache Event-Archiv (EventRecord) genutzt.

> **Transaktionsgrenzen:** Pro-Pipeline-Services sind POJOs ohne Spring-AOP-Proxy — `@Transactional` ist daher nicht verfuegbar. Jeder einzelne Spring-Data-JPA-Repository-Aufruf laeuft in einer eigenen Transaktion (Default-Verhalten). Fuer atomare Multi-Statement-Operationen (z.B. Trade + Fill in einem Commit) erhaelt der betreffende Service ein `TransactionTemplate` via PipelineFactory und wickelt die Aufrufe programmatisch ab.

Jede Pipeline erhaelt ihre eigene Konfiguration via `odin.agent.pipelines.{name}.*` Properties.

---

## 8. Konfigurationsarchitektur (Import-Kaskade)

```
odin-app/src/main/resources/
├── application.properties              ← Einstiegspunkt
│   └── spring.config.import=
│       └── optional:classpath:odin-core.properties
│
odin-core/src/main/resources/
├── odin-core.properties                ← Core-Defaults
│   └── spring.config.import=
│       ├── optional:classpath:odin-broker.properties
│       ├── optional:classpath:odin-data.properties
│       ├── optional:classpath:odin-brain.properties
│       ├── optional:classpath:odin-execution.properties
│       ├── optional:classpath:odin-audit.properties
│       └── optional:classpath:odin-persistence.properties
│
odin-{modul}/src/main/resources/
├── odin-{modul}.properties             ← Modul-Defaults
```

**Regeln:**
- Jedes Modul definiert seine Defaults in `odin-{modul}.properties`
- Import nur direkter Kinder (kein Aufwaerts-Import, keine Zyklen)
- Overwrites auf Schluessselebene in `application.properties` oder via ENV/JVM
- Properties-Klassen als Records mit `@ConfigurationProperties` + `@Validated`
- Secrets (`odin.broker.api-secret`, `odin.llm.api-key`) ausschliesslich via ENV/JVM, nie in Properties-Dateien

---

## 9. Querschnittliche Abhaengigkeiten

### Domain-spezifische Libraries (gekapselt in einem Modul)

| Abhaengigkeit | Genutzt von | Scope |
|---------------|-------------|-------|
| TWS API 10.40 | odin-broker | Einziges Modul mit Broker-Abhaengigkeit |
| ta4j | odin-brain | Einziges Modul mit ta4j-Abhaengigkeit |
| Claude SDK | odin-brain | LLM-Provider-Adapter (nur in `de.its.odin.brain.live`) |
| OpenAI SDK | odin-brain | LLM-Provider-Adapter (nur in `de.its.odin.brain.live`) |

**Prinzip (Domain-spezifisch):** Domain-spezifische Libraries bleiben in genau einem Modul gekapselt. Kein ta4j-Import in odin-execution, kein TWS-API-Import in odin-brain.

### Infrastruktur-Libraries (querschnittlich nutzbar)

| Abhaengigkeit | Genutzt von | Begruendung |
|---------------|-------------|-------------|
| Spring Data JPA | brain, execution, core, audit (via odin-persistence) | JPA ist Infrastruktur — jedes persistierende Domaenen-Modul benoetigt Entity-Annotations und Repository-Interfaces |
| PostgreSQL Driver | odin-persistence (transitiv) | JDBC-Treiber, zentral konfiguriert |
| Flyway | odin-persistence | Schema-Migration, zentral verwaltet |
| Spring Boot Starter Web | odin-app | REST-Endpoints, SSE |
| Micrometer | odin-core, odin-app | Metriken |

**Prinzip (Infrastruktur):** Infrastruktur-Libraries (JPA, Logging, Validation) duerfen in mehreren Modulen genutzt werden. Sie sind keine fachliche Kopplung — jedes Modul nutzt sie fuer seine eigene Domaene. Die technische Bereitstellung (DataSource, EntityManagerFactory) erfolgt zentral in odin-persistence.

---

## 10. Offene Punkte (Kapitel 1)

| # | Frage | Status |
|---|-------|--------|
| M1 | Base-Package: `de.its.odin` | ENTSCHIEDEN — `de.its.odin` |
| M2 | Frontend-Build: In Maven integriert (frontend-maven-plugin) oder komplett separater Build? | OFFEN |
| M3 | EventLog: Braucht es spaeter eine File-basierte Alternative (z.B. fuer lokale Analyse ohne DB)? | OFFEN (v2) |

---

## 11. Persistierung: Zwei Schreibpfade

### 11.1 Direkter Domänen-Schreibpfad (strukturierte Daten)

Domaenen-Module persistieren ihre strukturierten Daten **direkt und synchron** ueber eigene Repositories:

```
OMS empfaengt BrokerEvent (Fill)
  ├── OMS schreibt Trade/Fill ueber TradeRepository/FillRepository (direkt, synchron)
  └── OMS emittiert FillEvent an EventLog-Port
        └── AuditEventDrainer (odin-audit) schreibt EventRecord (async/sync je Modus)
```

| Domaene | Schreibt direkt | Ueber |
|---------|----------------|-------|
| odin-execution (OMS) | Trade, Fill, TradingRun, ConfigSnapshot | TradeRepository, FillRepository, TradingRunRepository, ConfigSnapshotRepository |
| odin-brain (DecisionArbiter, LLM-Clients) | DecisionLog, LlmCall | DecisionLogRepository, LlmCallRepository |
| odin-core (LifecycleManager) | PipelineState, DailyPerformance | PipelineStateRepository, DailyPerformanceRepository |

> **Kein PersistenceEventDrainer mehr.** Die alte Architektur hatte einen zentralen Drainer der aus Events sowohl EventRecords als auch strukturierte Tabellen befuellte. Im neuen Modell persistieren die Domaenen direkt — der AuditEventDrainer ist nur noch fuer das flache Event-Archiv (EventRecord) zustaendig.

### 11.2 EventLog-Schreibpfad (flaches Archiv)

Der EventLog bleibt zentral fuer Record/Replay (Kap 0, Abschnitt 3.1). Der `AuditEventDrainer` in odin-audit schreibt **ausschliesslich EventRecords** (flaches Archiv). Die Schreibstrategie unterscheidet sich je nach Modus:

| Modus | Schreibverhalten | Begruendung |
|-------|-----------------|-------------|
| **Simulation** | **Synchron** — EventLog-Schreiben ist in die Bar-Close-Barrier eingebunden. Erst wenn alle Events des aktuellen Cycles persistiert sind, gibt der Runner das naechste Event frei | Determinismus: Replay braucht vollstaendige, geordnete Event-Sequenz |
| **Live** | **Asynchron** — Events werden gepuffert und in Batches geschrieben. Reihenfolge wird ueber `sequenceNumber` und `eventId` garantiert und ist im Replay rekonstruierbar | Performance: DB-Writes duerfen den kritischen Pfad nicht blockieren |

> **Replay-Safety:** Unabhaengig vom Modus muss jedes Event eine monotone `sequenceNumber` pro Pipeline und einen globalen `eventId` tragen. Damit ist die Reihenfolge auch bei asynchronem Schreiben rekonstruierbar.

**Backpressure-Policy (Live):** Der asynchrone AuditEventDrainer darf bei Ueberlast High-Frequency MarketEvents sampeln oder droppen. **BrokerEvents, Decisions, LLM-Outputs und ConfigSnapshot-Referenzen sind non-droppable** — fuer diese Eventklassen gilt Backpressure bis geschrieben (bounded Queue). Laeuft die bounded Queue voll: Kill-Switch (fail-fast statt stiller Datenverlust).
