# ODIN — Guardrail: Maven-Modul-Strukturkonvention

Version: 1.2
Stand: 2026-02-15
Review: ChatGPT R1. DDD-Modulschnitt: odin-persistence aufgeloest, Entities in Domaenen-Module, odin-audit neu, Library-Regel angepasst

---

## 1. Zweck

Dieses Dokument definiert die einheitliche Basisstruktur fuer alle Maven-Module im ODIN-Projekt. Es ist pragmatisch gehalten — kein Dogma, aber ein verbindlicher Konsens. Ausnahmen sind erlaubt und werden, wo typisch, explizit benannt.

**Geltungsbereich:** 9 Java-Module (odin-api, odin-broker, odin-data, odin-brain, odin-execution, odin-core, odin-audit, odin-persistence, odin-app). Das 10. Modul — odin-frontend (React) — hat einen separaten Build und ist nicht Gegenstand dieses Guardrails.

**Referenzen:**
- Kapitel 0: Systemuebersicht (Architekturprinzipien, Port-Definitionen)
- Kapitel 1: Modularchitektur (Abhaengigkeitsgraph, Package-Konvention, Instanziierungsmodell)
- Kapitel 8: Datenmodell (Entity-Struktur, Persistence-Architektur)

---

## 2. Package-Struktur

### 2.1 Base-Package

**Base-Package:** `de.its.odin`

Jedes Modul erhaelt ein dediziertes Sub-Package:

| Modul | Root-Package |
|-------|-------------|
| odin-api | `de.its.odin.api` |
| odin-broker | `de.its.odin.broker` |
| odin-data | `de.its.odin.data` |
| odin-brain | `de.its.odin.brain` |
| odin-execution | `de.its.odin.execution` |
| odin-core | `de.its.odin.core` |
| odin-audit | `de.its.odin.audit` |
| odin-persistence | `de.its.odin.persistence` |
| odin-app | `de.its.odin.app` |

Kein Modul darf Klassen in das Package eines anderen Moduls legen. Die Root-Packages sind disjunkt.

### 2.2 Interne Sub-Package-Struktur (Standard)

Jedes Fachmodul (broker, data, brain, execution) und odin-core organisiert sich intern nach folgendem Schema:

```
de.its.odin.{modul}/
├── config/         # @ConfigurationProperties Records (+ @Configuration nur in Singleton-Modulen)
├── model/          # Modul-interne Datentypen (Records, Enums, Value Objects)
├── service/        # Geschaeftslogik (Services, Engines) — NICHT Spring @Service bei Pro-Pipeline-Modulen
├── persistence/    # Entity + Repository (nur in persistierenden Domaenen-Modulen: brain, execution, core)
│   ├── entity/     # JPA Entities ({Name}Entity)
│   └── repository/ # Spring Data JPA Repositories ({Name}Repository)
├── exception/      # Modul-spezifische Exceptions
├── live/           # Produktive Implementierungen (nicht IB-spezifisch)
├── sim/            # Simulation-Implementierungen
└── ib/             # IB-spezifischer Code (nur in odin-broker, zaehlt als Live-Variante)
```

**Konventionen:**
- `config/` enthaelt Konfigurationsinfrastruktur. In **Singleton-Modulen** (broker, persistence, core): `@ConfigurationProperties` Records und `@Configuration` Bean-Definitionen. In **Pro-Pipeline-Modulen** (data, brain, execution): **nur** `@ConfigurationProperties` Records — keine Bean-Definitionen, keine @Configuration-Klassen (Properties werden zentral in odin-app aktiviert).
- `model/` enthaelt nur Datentypen ohne Seiteneffekte (Records, Enums). Kein Spring-Kontext.
- `service/` enthaelt die Kernlogik des Moduls. In Pro-Pipeline-Modulen sind das plain Java Objects (keine Spring-Annotations), in Singleton-Modulen duerfen `@Service`/`@Component` verwendet werden. Services duerfen modul-intern andere Services aufrufen.
- `live/` und `sim/` trennen Port-Implementierungen nach Laufzeitmodus. Ein Package darf nicht in das andere importieren.
- `ib/` ist exklusiv fuer odin-broker reserviert. Es ist eine Spezialisierung von `live/` — der gesamte IB-TWS-spezifische Code liegt hier (z.B. `IbBrokerGateway`, `IbSession`, `IbDispatcher`). odin-broker hat `ib/` statt `live/`, weil IB die einzige Live-Variante ist. Das `live/` Package darf in odin-broker fehlen.
- `persistence/` enthaelt JPA Entities und Spring Data Repositories fuer persistierende Domaenen-Module (brain, execution, core). Entities heissen `{Name}Entity`, Repositories `{Name}Repository`. Das `persistence/` Package wird per `@EnableJpaRepositories` und `@EntityScan` in odin-app aktiviert. Repositories sind Singletons (Spring Beans) und werden Pro-Pipeline-Services per Konstruktor uebergeben.
- `exception/` ist optional — nur anlegen wenn modulspezifische Exceptions existieren.

**Erlaubte Abweichungen:**
- Sub-Domains innerhalb eines Moduls duerfen eigene Unter-Packages haben (z.B. `de.its.odin.brain.rules/`, `de.its.odin.brain.kpi/`). In diesem Fall wird die Standard-Struktur eine Ebene tiefer repliziert, sofern noetig.
- Module ohne Live/Sim-Trennung (z.B. odin-persistence) brauchen kein `live/` oder `sim/` Package.

### 2.3 odin-api Package-Struktur (Sonderrolle)

odin-api ist das einzige Modul mit einer abweichenden Struktur, da es ausschliesslich Vertraege enthaelt:

```
de.its.odin.api/
├── port/           # Zentrale Interfaces (MarketClock, MarketDataFeed, BrokerGateway, LlmAnalyst, EventLog)
├── event/          # Event-Records (MarketEvent, BrokerEvent, LlmAnalysis, RunEvent)
├── dto/            # Shared DTOs (MarketSnapshot, TradeIntent, QuantScore)
├── model/          # Shared Enums und Value Objects (PipelineState, Regime, RuntimeMode, ...)
└── context/        # RunContext
```

**Strikte Regeln fuer odin-api:**
- Kein Spring-Kontext (keine Annotations wie @Component, @Configuration)
- Keine Implementierungen — nur Interfaces, Records, Enums
- Keine externen Abhaengigkeiten (ausser JDK und jakarta.validation-api)
- Kein `config/`, kein `service/`, kein `exception/` Package
- **Nullability:** Wird ueber Bean Validation (`@NotNull`), `Optional` und JavaDoc geregelt — keine zusaetzlichen Annotations-Libraries (kein `@Nullable` aus Spring, Jetbrains, etc.)

### 2.4 odin-persistence Package-Struktur (Infrastruktur)

```
de.its.odin.persistence/
└── config/         # DataSourceConfiguration, JPA-Setup, Flyway-Config
```

**Konvention:** odin-persistence enthaelt **keine Entities, keine Repositories, keine Geschaeftslogik**. Es ist ein reines Infrastruktur-Modul fuer DataSource, EntityManagerFactory, TransactionManager und Schema-Migration.

### 2.4a odin-audit Package-Struktur

```
de.its.odin.audit/
├── config/         # AuditProperties
├── entity/         # EventRecordEntity
├── repository/     # EventRecordRepository
└── service/        # PostgresEventLog (implements EventLog Port), AuditEventDrainer
```

### 2.4b Persistenz in Domaenen-Modulen

Domaenen-Module mit Entities nutzen ein `persistence/` Sub-Package:

```
de.its.odin.execution/
├── ...bestehende Packages...
└── persistence/
    ├── entity/     # TradingRunEntity, TradeEntity, FillEntity, ConfigSnapshotEntity
    └── repository/ # TradingRunRepository, TradeRepository, FillRepository, ConfigSnapshotRepository

de.its.odin.brain/
├── ...bestehende Packages...
└── persistence/
    ├── entity/     # LlmCallEntity, DecisionLogEntity
    └── repository/ # LlmCallRepository, DecisionLogRepository

de.its.odin.core/
├── ...bestehende Packages...
└── persistence/
    ├── entity/     # PipelineStateEntity, DailyPerformanceEntity
    └── repository/ # PipelineStateRepository, DailyPerformanceRepository
```

**Konventionen (fuer alle persistierenden Module):**
- Entities heissen `{Name}Entity` (z.B. `TradeEntity`)
- Repositories heissen `{Name}Repository` (z.B. `TradeRepository`)
- `@Embeddable` Value Types heissen `{Name}Embeddable` (z.B. `CostBreakdownEmbeddable`)
- Entities sind keine DTOs und werden nicht moduluebergreifend als Datentraeger verwendet
- Repositories sind Singletons (Spring Data JPA) — aktiviert via `@EnableJpaRepositories` in odin-app

### 2.5 odin-app Package-Struktur

```
de.its.odin.app/
├── config/         # Mode-Wiring (LiveWiringConfig, SimWiringConfig), EnableConfigurationProperties
├── controller/     # REST Controller, SSE Endpoints
└── OdinApplication.java
```

odin-app ist die Composition Root. Hier werden Port-Implementierungen an Ports gebunden (Mode-abhängig) und alle @EnableConfigurationProperties aktiviert.

---

## 3. Namenskonventionen

### 3.1 Klassen

| Typ | Namensschema | Beispiel |
|-----|-------------|---------|
| Service (Geschaeftslogik) | `{Fachbegriff}Service` | `DataPipelineService`, `OrderManagementService` |
| Engine (regelbasiert, zustandsbehaftet) | `{Fachbegriff}Engine` | `KpiEngine`, `RulesEngine` |
| Port-Interface | Sprechender Name ohne Suffix | `MarketClock`, `BrokerGateway`, `LlmAnalyst` |
| Port-Implementierung (Live) | `{Technologie}{Port}` oder `System{Port}` | `IbBrokerGateway`, `ClaudeAnalystClient`, `SystemMarketClock` |
| Port-Implementierung (Sim) | `{Sim-Konzept}` oder `{Konzept}Simulator` | `SimClock`, `BrokerSimulator`, `CachedAnalyst` |
| Adapter (Datenquelle) | `{Quelle}{Domäne}Adapter` | `IbAccountAdapter` |
| DTO (shared, in odin-api) | Sprechender Name ohne Suffix | `MarketSnapshot`, `TradeIntent`, `QuantScore` |
| Event (in odin-api) | `{Domäne}Event` | `MarketEvent`, `BrokerEvent` |
| Entity (JPA, im jeweiligen Domaenen-Modul) | `{Name}Entity` | `TradeEntity`, `FillEntity` |
| Repository | `{Entity-Stammname}Repository` | `TradeRepository`, `FillRepository` |
| Properties (Konfiguration) | `{Modul/Domäne}Properties` | `BrokerProperties`, `DataPipelineProperties` |
| Configuration (@Configuration) | `{Zweck}Configuration` oder `{Zweck}Config` | `LiveWiringConfig`, `SimWiringConfig` |
| Enum | Fachlicher Name (Singular) | `Regime`, `PipelineState`, `OrderType` |
| Exception | `{Fachbegriff}Exception` | `StaleDataException`, `RiskLimitExceededException` |
| Factory | `{Produkt}Factory` | `PipelineFactory`, `MarketSnapshotFactory` |
| Validator/Gate | `{Fachbegriff}Gate` oder `{Fachbegriff}Validation` | `DataQualityGate`, `RiskGate`, `QuantValidation` |

**Verboten:**
- Suffix `Impl` (stattdessen sprechende Namen: `IbBrokerGateway`, nicht `BrokerGatewayImpl`)
- Generische Namen ohne Kontext (`Manager`, `Helper`, `Utils`) — Ausnahme: `GlobalRiskManager` ist akzeptiert, weil `Global` den Kontext liefert
- Abkuerzungen im Klassennamen (ausser etablierte Domänen-Abkuerzungen: `Ib`, `Llm`, `Kpi`, `Oms`, `Dq`)

### 3.2 Packages

| Zweck | Package-Name | Verboten |
|-------|-------------|----------|
| Konfiguration | `config` | `configuration`, `cfg`, `props` |
| Geschaeftslogik | `service` | `services`, `svc`, `logic` |
| Datentypen | `model` | `models`, `domain`, `types`, `dto` (ausser in odin-api) |
| Domaenen-Persistenz | `persistence` | `persist`, `db`, `jpa` |
| JPA Entities | `entity` (unter `persistence/` oder direkt in odin-audit) | `entities`, `model`, `domain` |
| Spring Data Repos | `repository` (unter `persistence/` oder direkt in odin-audit) | `repositories`, `repo`, `repos`, `dao` |
| REST Endpoints | `controller` | `controllers`, `rest`, `web`, `api` (in odin-app) |
| Exceptions | `exception` | `exceptions`, `error`, `errors` |
| IB-spezifisch | `ib` | `ibgateway`, `tws`, `interactivebrokers` |
| Live-Implementierung | `live` | `prod`, `production`, `real` |
| Simulation | `sim` | `simulation`, `test`, `mock` |

### 3.3 Methoden

Methoden benennen sich nach dem, was sie tun (Verb-first):

| Muster | Beispiel |
|--------|---------|
| Aktion ausfuehren | `submitOrder()`, `calculateQuantScore()`, `buildBar()` |
| Abfrage/Laden | `loadOpenTrades()`, `findByRunId()`, `resolveRegime()` |
| Prüfung (Boolean) | `isStale()`, `hasOpenPosition()`, `canTrade()` |
| Erzeugen | `createPipeline()`, `createSnapshot()` |
| Transformation | `toEntity()`, `fromEvent()` |
| Lifecycle | `start()`, `stop()`, `initialize()` |

---

## 4. Konfigurationskonzept

### 4.1 Properties-Dateien

Jedes Modul definiert seine Defaults in einer eigenen Properties-Datei:

| Modul | Properties-Datei | Pfad |
|-------|-----------------|------|
| odin-broker | `odin-broker.properties` | `odin-broker/src/main/resources/` |
| odin-data | `odin-data.properties` | `odin-data/src/main/resources/` |
| odin-brain | `odin-brain.properties` | `odin-brain/src/main/resources/` |
| odin-execution | `odin-execution.properties` | `odin-execution/src/main/resources/` |
| odin-core | `odin-core.properties` | `odin-core/src/main/resources/` |
| odin-audit | `odin-audit.properties` | `odin-audit/src/main/resources/` |
| odin-persistence | `odin-persistence.properties` | `odin-persistence/src/main/resources/` |
| odin-app | `application.properties` | `odin-app/src/main/resources/` |

**Regeln:**
- Alle Defaults stehen in der Properties-Datei, nicht in Java-Code
- Secrets (`odin.*.api-key`, `odin.*.api-secret`, `odin.*.password`) niemals in Properties-Dateien — ausschliesslich via ENV oder JVM-Systemproperties
- Properties-Dateien sollen dokumentiert sein (Kommentare ueber jeden logischen Block)

**odin-api hat keine Properties-Datei** (keine Konfiguration, keine Spring-Abhaengigkeit).

### 4.2 Namespace-Schema

```
odin.{modul}.{komponente}.{property}
```

Beispiele:
```properties
# odin-broker
odin.broker.ib.host=localhost
odin.broker.ib.port=7497
odin.broker.ib.client-id=1
odin.broker.ib.reconnect-delay-ms=5000
odin.broker.sim.slippage-bps=5.0
odin.broker.sim.fill-delay-ms=100

# odin-data
odin.data.buffer.capacity=500
odin.data.dq.stale-threshold-ms=30000
odin.data.dq.outlier-sigma=5.0

# odin-brain
odin.brain.kpi.parallel-timeframes=true
odin.brain.llm.claude.model-id=claude-sonnet-4-5-20250929
odin.brain.llm.openai.model-id=gpt-4o

# odin-execution
odin.execution.risk.max-position-risk-percent=2.0
odin.execution.risk.max-daily-loss-percent=5.0
odin.execution.oms.tranchen-variant=STANDARD_4T

# odin-core
odin.core.pipeline.decision-interval-ms=60000
odin.core.lifecycle.pre-market-offset-minutes=15
odin.core.sim.speed-mode=DECISION_SYNCED

# odin-persistence
odin.persistence.datasource.url=jdbc:postgresql://localhost:5432/odindb
odin.persistence.drain.interval-ms=500
odin.persistence.drain.batch-size=100
odin.persistence.retention.sim-run-days=90
```

**Konventionen:**
- Kebab-Case fuer Property-Keys (Spring-Standard)
- Zeitwerte mit Einheiten-Suffix: `-ms`, `-s`, `-minutes`
- Groessenwerte mit Einheiten-Suffix: `-bytes`, `-mb`
- Prozente mit Suffix: `-percent`
- Booleans: sprechend benennen, Prefix `enable-` oder `is-` bevorzugt (z.B. `enable-parallel-timeframes`). Bei eindeutigem Kontext darf der Bool-Prefix entfallen

### 4.3 Import-Kaskade

```
application.properties (odin-app)
  └── spring.config.import=optional:classpath:odin-core.properties
        └── spring.config.import=
              ├── optional:classpath:odin-broker.properties
              ├── optional:classpath:odin-data.properties
              ├── optional:classpath:odin-brain.properties
              ├── optional:classpath:odin-execution.properties
              ├── optional:classpath:odin-audit.properties
              └── optional:classpath:odin-persistence.properties
```

**Regeln:**
- Import nur direkter Kinder (kein Aufwaerts-Import, keine Zyklen)
- `optional:` Prefix obligatorisch (verhindert Startfehler bei fehlender Datei)
- Overwrites auf Schluesselebene in `application.properties` oder via ENV/JVM
- Prioritaet (hoechste zuerst): JVM/ENV > application.properties > odin-core.properties > odin-{modul}.properties

### 4.4 @ConfigurationProperties Records

Jedes Modul, das konfigurierbar ist, stellt mindestens einen Properties-Record bereit:

```java
@ConfigurationProperties(prefix = "odin.broker")
@Validated
public record BrokerProperties(
        @NotNull @Valid IbProperties ib,
        @NotNull @Valid SimProperties sim
) {
    // Nested Records fuer Sub-Namespaces
    @Validated
    public record IbProperties(
            @NotBlank String host,
            @Min(1) int port,
            @Min(0) int clientId,
            @Min(100) long reconnectDelayMs
    ) { }

    @Validated
    public record SimProperties(
            @DecimalMin("0.0") double slippageBps,
            @Min(0) long fillDelayMs
    ) { }
}
```

**Regeln:**
- Immer als `record` (Java 21), nie als Klasse
- Immer `@Validated` und Bean Validation Annotations auf Pflichtfeldern
- Kein `@Value`-basiertes Property-Binding — ausschliesslich `@ConfigurationProperties`
- Keine Java-Defaults (Default-Werte stehen in der Properties-Datei)
- Nested Records fuer Sub-Namespaces sind erlaubt und empfohlen
- Properties-Records liegen im `config/` Package des jeweiligen Moduls

### 4.5 Aktivierung

Die zentrale Aktivierung aller Properties erfolgt in odin-app:

```java
@EnableConfigurationProperties({
        BrokerProperties.class,
        DataPipelineProperties.class,
        BrainProperties.class,
        ExecutionProperties.class,
        CoreProperties.class,
        AuditProperties.class,
        PersistenceProperties.class
})
```

Kein Fachmodul aktiviert seine eigenen Properties. Das ist Aufgabe der Composition Root (odin-app).

---

## 5. API-Design

### 5.1 Shared API (odin-api)

In odin-api gehoert alles, was moduluebergreifend benoetigt wird:

| Typ | Package | Beispiel |
|-----|---------|---------|
| Port-Interfaces | `de.its.odin.api.port` | `MarketClock`, `BrokerGateway` |
| Cross-Module Events | `de.its.odin.api.event` | `MarketEvent`, `BrokerEvent` |
| Cross-Module DTOs | `de.its.odin.api.dto` | `MarketSnapshot`, `TradeIntent` |
| Shared Enums | `de.its.odin.api.model` | `Regime`, `PipelineState`, `OrderType` |
| RunContext | `de.its.odin.api.context` | `RunContext` |

**Kriterium fuer Aufnahme in odin-api:** Der Typ wird von mindestens zwei Modulen benoetigt ODER ist Teil eines Port-Vertrags.

### 5.2 Modul-interne API

Typen, die nur innerhalb eines Moduls verwendet werden, bleiben im `model/` Package des Moduls:

```
de.its.odin.brain.model/
├── KpiResult.java          # Nur innerhalb odin-brain genutzt
├── RulesState.java         # Interner Zustand der RulesEngine
└── AnalystPrompt.java      # LLM-Prompt-Struktur
```

**Faustregel:** Wenn ein Typ nur ein einziger Service benoetigt, gehoert er nicht nach odin-api. Wenn spaeter ein zweites Modul ihn braucht, wird er migriert.

### 5.3 Port-Interface-Konventionen

Port-Interfaces (in `de.its.odin.api.port`) folgen diesen Regeln:

- Reine Methoden-Vertraege, keine Default-Implementierungen
- Parameter und Rueckgabewerte sind odin-api-Typen (DTOs, Events, Enums) oder JDK-Standardtypen
- Keine Spring-Annotations auf dem Interface
- Ports sollen eine Live- und eine Sim-Implementierung haben. Wenn die Implementierung in beiden Modi identisch ist (wie EventLog/PostgreSQL), genuegt eine einzige — die Entscheidung muss dokumentiert sein
- JavaDoc dokumentiert den Vertrag, Vorbedingungen und Verhalten bei Fehlern

### 5.4 Event-Konventionen

Events (in `de.its.odin.api.event`) sind immutable Records:

- Jedes Event traegt `runId`, `instrumentId`, `marketClockTimestamp` und `sequenceNumber`
- Events werden nie mutiert — bei Statusaenderungen wird ein neues Event erzeugt
- Event-Typen repraesentieren fachliche Ereignisse, keine technischen (kein `DatabaseWriteEvent`)
- Events werden ueber den EventLog-Port emittiert, nie direkt zwischen Modulen weitergereicht

---

## 6. Dependency-Regeln

### 6.1 Abhaengigkeitsgraph (normativ)

Pfeile zeigen die "depends-on"-Richtung (A → B = A haengt von B ab):

```
                            odin-api                        (keine Abhaengigkeiten)
                               ▲
                               │ depends-on
       ┌───────┬──────────────┼──────────────┬──────────┬──────────┐
       │       │              │              │          │          │
   broker    data          brain        execution    audit        │
       ▲       ▲              ▲              ▲          ▲          │
       │       │              │              │          │          │
       └───────┴──────────────┴──────────────┴──────────┘          │
                              │ depends-on                          │
                          odin-core ────────────────────────────────┘
                              ▲
                              │ depends-on
                          odin-app

    odin-persistence (Infrastruktur):
      → Transitive Dependency fuer brain, execution, core, audit
      → Liefert: DataSource-Config, JPA, PostgreSQL-Driver
```

Fachmodule zeigen nur nach oben (zu odin-api). Persistierende Fachmodule (brain, execution, core, audit) haben zusaetzlich eine Dependency auf odin-persistence (Infrastruktur). odin-core zeigt nach oben (zu den Fachmodulen und odin-api). odin-app zeigt nach oben (zu odin-core). Keine seitlichen oder abwaerts gerichteten Abhaengigkeiten.

### 6.2 Regeln (strikt)

| Regel | Erlaubt | Verboten |
|-------|---------|----------|
| Fachmodule → odin-api | Ja | |
| Fachmodule → Fachmodule | | Ja (kein brain→broker, kein execution→data, kein audit→brain) |
| Fachmodule → odin-core | | Ja |
| Fachmodule → odin-app | | Ja |
| Persistierende Fachmodule → odin-persistence | Ja (Infrastruktur) | |
| odin-core → alle Fachmodule | Ja | |
| odin-core → odin-api | Ja | |
| odin-app → odin-core | Ja (transitiv alle Module) | |
| odin-api → irgendein Modul | | Ja (keine Abhaengigkeiten) |
| odin-persistence → Fachmodule | | Ja (kennt keine fachlichen Entities) |

### 6.3 Externe Libraries

**Domain-spezifische Libraries** sind in genau einem Modul gekapselt:

| Library | Einziges erlaubtes Modul |
|---------|------------------------|
| TWS API 10.40 | odin-broker |
| ta4j | odin-brain |
| Claude SDK | odin-brain |
| OpenAI SDK | odin-brain |
| Spring Boot Starter Web | odin-app |

Kein anderes Modul darf diese Libraries direkt importieren. Die Kommunikation erfolgt ueber die Port-Interfaces in odin-api.

**Infrastruktur-Libraries** duerfen in mehreren Modulen genutzt werden:

| Library | Genutzt von | Begruendung |
|---------|-------------|-------------|
| Spring Data JPA | brain, execution, core, audit (via odin-persistence) | Jedes persistierende Domaenen-Modul benoetigt Entity-Annotations und Repository-Interfaces |
| PostgreSQL Driver | odin-persistence (transitiv) | Zentral konfiguriert |
| Flyway | odin-persistence | Schema-Migration |
| Micrometer | odin-core, odin-app | Metriken |

> **Unterscheidung:** Domain-spezifische Libraries (TWS API, ta4j, LLM-SDKs) repraesentieren fachliche Kapselung — sie gehoeren in genau ein Modul. Infrastruktur-Libraries (JPA, Logging, Validation) sind technische Querschnittsanliegen — ihre Nutzung in mehreren Modulen stellt keine fachliche Kopplung dar.

### 6.4 Enforcement

- **ArchUnit-Tests:** Automatisierte Pruefung der Package- und Layer-Regeln:
  - Fachmodule duerfen nicht voneinander abhaengen
  - `live/` und `sim/` Packages duerfen nicht gegenseitig importieren
  - `ib/` darf nicht `sim/` importieren (und umgekehrt)
  - Pro-Pipeline-Module duerfen keine `@Component`/`@Service` Annotations verwenden (Ausnahme: `@Repository` fuer Spring Data JPA Interfaces in `persistence/` Sub-Packages ist erlaubt — diese sind Singletons)
- **Maven Enforcer Plugin:** Verbotene transitive Dependencies in falschen Modulen erkennen

---

## 7. Instanziierungsmodell & Spring-Kontext

### 7.1 Singleton vs. Pro-Pipeline

ODIN unterscheidet zwischen Singleton-Komponenten (ein Mal pro JVM) und Pro-Pipeline-Komponenten (je Instrument):

| Scope | Module | Spring Bean? |
|-------|--------|-------------|
| Singleton | odin-broker (IbSession, IbDispatcher), odin-audit (PostgresEventLog, AuditEventDrainer), odin-persistence (DataSource), odin-core (LifecycleManager, GlobalRiskManager), odin-app | Ja (@Component, @Service) |
| Singleton (Repositories) | brain, execution, core, audit (Spring Data JPA Interfaces) | Ja (@Repository via @EnableJpaRepositories) |
| Pro-Pipeline | odin-data, odin-brain, odin-execution (Services), odin-broker (Facades), odin-core (PipelineStateMachine) | Nein (PipelineFactory erzeugt manuell, Repositories per Konstruktor) |

### 7.2 Component-Scan

Component-Scan ist beschraenkt auf:
- `de.its.odin.app`
- `de.its.odin.core`
- `de.its.odin.broker`
- `de.its.odin.audit`

Zusaetzlich werden Repositories via `@EnableJpaRepositories` und Entities via `@EntityScan` aktiviert (nicht ueber Component-Scan):
- `de.its.odin.execution.persistence.repository` / `.entity`
- `de.its.odin.brain.persistence.repository` / `.entity`
- `de.its.odin.core.persistence.repository` / `.entity`
- `de.its.odin.audit.repository` / `.entity`

`odin-persistence` wird **nicht** vom Component-Scan erfasst — es stellt nur `@Configuration`-Klassen bereit (DataSourceConfiguration), die ueber `@Import` oder Boot-Auto-Config geladen werden.

Pro-Pipeline-Module (data, brain, execution) werden NICHT vom Component-Scan erfasst. Die `PipelineFactory` in odin-core erzeugt deren Service-Instanzen als plain Java Objects und verdrahtet sie gegen Port-Interfaces. **Repositories** in diesen Modulen sind davon ausgenommen — sie sind Singletons und werden per `@EnableJpaRepositories` aktiviert.

**Wichtig:** Pro-Pipeline-Module koennen dennoch Klassen enthalten, die als Singletons genutzt werden (z.B. `ClaudeAnalystClient` in odin-brain). Diese werden **nicht** per Component-Scan gefunden, sondern ausschliesslich in odin-app per `@Bean`-Methode instanziiert. In den Pro-Pipeline-Modulen sind `@Component` und `@Service` Annotations verboten. Spring Data JPA `@Repository`-Interfaces im `persistence/` Sub-Package sind die einzige Ausnahme (Singletons).

### 7.2a Wiring-Regel: Wer erzeugt was?

| Komponente | Erzeugt durch | Uebergabe an Pipeline |
|-----------|--------------|----------------------|
| Singleton-Adapter (IbSession, ClaudeAnalystClient, PostgresEventLog) | odin-app (Mode-Wiring: LiveWiringConfig/SimWiringConfig) | Als Port-Interface an PipelineFactory |
| Singleton-Services (GlobalRiskManager, LifecycleManager) | odin-core (@Configuration) | Direkt verwendet, nicht pro Pipeline |
| Singleton-Repositories (TradeRepository, LlmCallRepository, ...) | Spring Data JPA (@EnableJpaRepositories) | Per Konstruktor an PipelineFactory, weiter an Pipeline-Services |
| Pro-Pipeline-Objekte (DataPipelineService, KpiEngine, RiskGate, ...) | PipelineFactory in odin-core | Per Konstruktor mit Ports, Properties und Repositories |

**Klarstellung:** Singleton-Adapter wie `ClaudeAnalystClient` oder `IbBrokerGateway` werden in odin-app als Spring Beans erzeugt und als Port-Interfaces (z.B. `LlmAnalyst`, `BrokerGateway`) an die `PipelineFactory` weitergegeben. Repositories sind ebenfalls Singletons und werden per Constructor Injection an die PipelineFactory uebergeben. Die Factory erzeugt dann pro Pipeline die internen Komponenten und verbindet sie mit den Singleton-Ports und -Repositories. Pro-Pipeline-Klassen sind POJOs und erhalten ihre Abhaengigkeiten ausschliesslich per Konstruktor.

### 7.3 @Configuration-Klassen

| Modul | @Configuration Bean-Definitionen? | @ConfigurationProperties Records? | Begruendung |
|-------|-----------------------------------|----------------------------------|-------------|
| odin-api | Nein | Nein | Kein Spring-Kontext |
| odin-broker | Ja | Ja | Singleton (IbSession, IbDispatcher) |
| odin-data | **Nein** | Ja | Pro-Pipeline — keine Bean-Definitionen |
| odin-brain | **Nein** | Ja | Pro-Pipeline — keine Bean-Definitionen |
| odin-execution | **Nein** | Ja | Pro-Pipeline — keine Bean-Definitionen |
| odin-core | Ja | Ja | Singleton (Lifecycle, GlobalRisk, PipelineFactory) |
| odin-audit | Ja | Ja | Singleton (PostgresEventLog, AuditEventDrainer) |
| odin-persistence | Ja | Ja | Singleton (DataSource, JPA-Infra, Flyway) |
| odin-app | Ja | Nein (aktiviert fremde) | Composition Root, Mode-Wiring, @EnableJpaRepositories, @EntityScan |

**Wichtig:** Pro-Pipeline-Module (data, brain, execution) definieren `@ConfigurationProperties` Records in ihrem `config/` Package, aber **keine** `@Configuration`-Klassen mit Bean-Definitionen. Die Properties-Aktivierung erfolgt ausschliesslich in odin-app via `@EnableConfigurationProperties`. Die PipelineFactory erhaelt die gebundenen Properties per Konstruktor-Injection und gibt sie an die manuell erzeugten Pipeline-Objekte weiter.

---

## 8. Test-Struktur

### 8.1 Verzeichnislayout

```
odin-{modul}/
├── src/main/java/...
├── src/main/resources/
│   └── odin-{modul}.properties
├── src/test/java/...                    # Spiegelt die main-Struktur
└── src/test/resources/
    └── odin-{modul}-test.properties     # Test-spezifische Overrides
```

Die Test-Packagestruktur spiegelt exakt die main-Packagestruktur.

### 8.2 Naming-Conventions

| Test-Typ | Naming-Pattern | Suffix | Beispiel |
|----------|---------------|--------|---------|
| Unit-Test | `{Klasse}Test` | `Test` | `KpiEngineTest`, `RiskGateTest` |
| Integrationstest | `{Klasse}IntegrationTest` | `IntegrationTest` | `BrokerPropertiesIntegrationTest` |
| Ende-zu-Ende-Test | `{Feature}E2ETest` | `E2ETest` | `TradingPipelineE2ETest` |

**Suffixe sind fuer Build-Steuerung relevant:** Maven Surefire laeuft auf `*Test`, Failsafe auf `*IntegrationTest` und `*E2ETest`. Dadurch koennen Integrationstests separat ausgefuehrt und im normalen Build uebersprungen werden.

### 8.3 Test-Properties

- Test-spezifische Property-Overrides in `odin-{modul}-test.properties`
- Test-Properties ueberschreiben nur die Keys, die abweichen muessen (z.B. kuerzere Timeouts, In-Memory-DB-URL)
- Keine Produktions-Secrets in Test-Properties
- Integrationstests, die Spring-Kontext starten, nutzen `@TestPropertySource(locations = "classpath:odin-{modul}-test.properties")`
- Tests gegen externe APIs (LLM, TWS) muessen per Maven-Profil oder JUnit-Tag deaktivierbar sein, damit CI ohne Secrets/Infrastruktur laeuft

### 8.4 Test-Scope nach Modul

| Modul | Unit-Tests | Integrationstests | Anmerkung |
|-------|-----------|-------------------|-----------|
| odin-api | Nein (nur Typen) | Nein | Keine Logik zum Testen |
| odin-broker | Ja (IbDispatcher-Logik) | Ja (IbSession gegen TWS Paper) | E2E nur mit laufendem TWS Gateway |
| odin-data | Ja (Buffer, DQ, BarBuilder) | Nein | Keine externen Abhaengigkeiten |
| odin-brain | Ja (Rules, KPI, Quant) | Ja (LLM-Clients gegen echte APIs) | LLM-Tests brauchen API-Keys, standardmaessig deaktiviert (Profil/Tag) |
| odin-execution | Ja (Risk, OMS, Tranchen) | Nein | Laeuft gegen Port-Mocks |
| odin-core | Ja (Pipeline-FSM, Lifecycle) | Ja (Pipeline-Integration) | Integrationstests mit SimClock |
| odin-persistence | Minimal (Config-Logik) | Ja (DataSource/Flyway gegen Testcontainer) | Infrastruktur-Startup-Tests |
| odin-audit | Ja (Drainer-Logik) | Ja (gegen Testcontainer/H2) | EventRecord-Persistierung + Drain-Logik |
| odin-app | Minimal | Ja (Kontext-Start, Wiring) | Prueft, dass der Kontext hochfaehrt |

---

## 9. Modul-Checkliste (bei Neuanlage oder Aenderung)

Verwende diese Checkliste beim Anlegen eines neuen Moduls oder bei strukturellen Aenderungen:

### Maven/POM
- [ ] artifactId folgt dem Schema `odin-{name}`
- [ ] groupId ist `de.its.odin`
- [ ] Parent-POM ist `odin` (Root)
- [ ] Dependency auf `odin-api` vorhanden (ausser bei odin-api selbst)
- [ ] Keine verbotenen Abhaengigkeiten (Fachmodul→Fachmodul, etc.)
- [ ] Externe Libraries nur im zustaendigen Modul deklariert

### Packages
- [ ] Root-Package ist `de.its.odin.{modul}`
- [ ] Sub-Packages folgen dem Standard (config/, model/, service/, ...)
- [ ] Kein Code in fremden Modul-Packages

### Konfiguration
- [ ] Properties-Datei `odin-{modul}.properties` vorhanden
- [ ] Namespace `odin.{modul}.*` — kein Prefix-Overlap mit anderen Modulen
- [ ] Alle Defaults in Properties-Datei (keine Java-Defaults)
- [ ] Properties-Record als `@ConfigurationProperties` + `@Validated` Record vorhanden
- [ ] Import in odin-core.properties eingetragen (application.properties importiert nur odin-core.properties)
- [ ] Aktivierung in odin-app `@EnableConfigurationProperties` eingetragen
- [ ] Secrets nicht in Properties-Dateien

### Persistenz (wenn Modul Entities hat)
- [ ] Entities in `persistence/entity/` Sub-Package (oder `entity/` bei odin-audit)
- [ ] Repositories in `persistence/repository/` Sub-Package (oder `repository/` bei odin-audit)
- [ ] `@EnableJpaRepositories` in odin-app um neues Repository-Package erweitert
- [ ] `@EntityScan` in odin-app um neues Entity-Package erweitert
- [ ] odin-persistence als Dependency vorhanden (transitive JPA-Infra)

### Wiring & Enforcement
- [ ] Component-Scan-Base-Packages in odin-app pruefen/erweitern (wenn Singleton-Modul)
- [ ] ArchUnit-Tests um neues Package/Modul erweitert (Dependency-Regeln enforcen)
- [ ] Maven-Enforcer-Regeln erweitert (wenn neue gekapselte Library hinzukommt)

### Tests
- [ ] Test-Verzeichnis spiegelt main-Struktur
- [ ] Test-Properties `odin-{modul}-test.properties` vorhanden (wenn noetig)
- [ ] Unit-Tests fuer Geschaeftslogik
- [ ] Integrationstests fuer Properties-Binding und Spring-Kontext (wenn relevant)
- [ ] Test-Naming folgt den Konventionen (*Test, *IntegrationTest, *E2ETest)

### Qualitaet
- [ ] JavaDoc auf allen public Klassen und Methoden (ausser Getter/Setter)
- [ ] Keine `var`-Nutzung (verboten)
- [ ] Keine Magic Numbers (private static final Konstanten)
- [ ] Sprechende Variablennamen (mehr als 1 Zeichen, Ausnahme: Schleifenzaehler)

---

## 10. Zusammenfassung der Ausnahmen

| Ausnahme | Begruendung | Betrifft |
|----------|-------------|----------|
| odin-api hat keine Standard-Sub-Packages | Nur Vertraege, kein Spring-Kontext, keine Logik | odin-api |
| `ib/` Package nur in odin-broker | IB-spezifischer Code ist modulgekapselt | odin-broker |
| Sub-Domain-Packages erlaubt | Komplexere Module (brain) brauchen fachliche Unterteilung | odin-brain, odin-core |
| Pro-Pipeline-Module haben kein @Component | Manuelle Instanziierung durch PipelineFactory | odin-data, odin-brain, odin-execution |
| odin-persistence hat keine Entities/Repos | Reines Infrastruktur-Modul (DDD-Modulschnitt) | odin-persistence |
| Domaenen-Module haben `persistence/` Sub-Package | JPA Entities + Repos leben in der fachlichen Domaene | odin-brain, odin-execution, odin-core |
| odin-audit hat `entity/` direkt (nicht unter `persistence/`) | Eigenstaendiges Modul, nicht Sub-Package eines anderen Moduls | odin-audit |
| Repositories als Singletons in Pro-Pipeline-Modulen | Spring Data JPA Interfaces sind keine Services — @Repository ist erlaubt | odin-brain, odin-execution |
| EventLog hat keine Sim-Variante | Gleiche Implementierung in beiden Modi (PostgreSQL), dokumentiert | odin-audit |
| Infrastruktur-Libraries in mehreren Modulen | JPA, Logging etc. sind keine fachliche Kopplung | brain, execution, core, audit |
| `ib/` statt `live/` in odin-broker | IB ist die einzige Live-Variante, `ib/` zaehlt als Live-Package | odin-broker |
