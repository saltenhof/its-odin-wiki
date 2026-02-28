# Story: ODIN-057 — Agent-Native Diagnostic Layer

## Metadaten

| Feld | Wert |
|------|------|
| **Titel** | Agent-Native Diagnostic Layer |
| **Module** | odin-api (Enums, DTOs), odin-core (DiagnosticCollector, Ringbuffer), odin-app (REST/SSE-Endpoints, Controller), odin-frontend (Diagnostics-Panel) |
| **Phase** | 2 |
| **Abhaengigkeiten** | Keine Vorlaeufer. Alle betroffenen Module existieren bereits. |
| **Geschaetzter Umfang** | XL |

---

## Kontext

### Ist-Zustand (Problem)

ODIN ist ein vollautomatischer Trading-Agent, der komplexe Entscheidungsketten pro Bar durchlaeuft: Data Pipeline, KPI-Engine, Rules Engine, DecisionArbiter, RiskGate, OMS. Wenn ein Trade nicht stattfindet oder ein unerwartetes Verhalten auftritt, fehlt jede Moeglichkeit fuer einen externen Agenten (Claude, Operator, Debugger), die interne Entscheidungskette nachzuvollziehen:

- **Kein Echtzeit-Zugang** zu Gate-Zustaenden, Regel-Evaluierungen, Arbiter-Votes oder KPI-Zwischenwerten
- **EventLog ist historisch** — es persistiert finale Entscheidungen, nicht den Weg dorthin
- **SSE-Streams** liefern UI-optimierte Summaries (PipelineState, IndicatorUpdate), nicht die innere Mechanik
- **Logging** ist unstrukturiert und nicht per API abrufbar — ein Agent kann Logfiles nicht sinnvoll parsen
- **Kein Steuerungsmechanismus** fuer die Datendichte: entweder alles oder nichts

### Soll-Zustand

Ein Agent Diagnostic Layer, der:

1. **Umfangreiche Schnittstellen** bietet, ueber die ein Agent (Claude, externer Debugger) andocken kann
2. **Feature-Toggles** bereitstellt, mit denen der Agent granular steuern kann, **was** auf den Schnittstellen exponiert wird — ob gar nichts, ob kleine, mittlere oder grosse Datensaetze, oder fuer bestimmte Subsysteme gezielt Daten exponiert werden
3. In der **Vollausbaustufe** maximal verbose und voll instrumentiert ist, aber auch die Moeglichkeit bietet, den Overhead auf **exakt null** zu reduzieren
4. Sowohl **backend-seitig** (REST, SSE) als auch **frontend-seitig** (Diagnostics-Panel) nutzbar ist

### Geschaeftswert

- **Agent-Autonomie:** Ein LLM-Agent kann Backtests und Live-Trading selbststaendig diagnostizieren, ohne den Operator zu benoetigen
- **Fehlereingrenzung:** Statt "kein Trade" → "Entry blockiert bei RulesEngine Gate 3: RSI 72.4 > rsi-max 70"
- **Iterationsgeschwindigkeit:** Parameter-Tuning und Strategie-Optimierung werden um Groessenordnungen schneller, weil der Feedback-Loop sichtbar wird
- **Zero-Cost-Default:** Im Produktivbetrieb (OFF) entsteht kein Overhead — der Layer ist architektonisch so angelegt, dass er zum Nulltarif deaktivierbar ist

---

## Architekturdiagramm

```
                        ┌──────────────────────────────────────────────────────────┐
                        │                   ODIN Backend                           │
                        │                                                          │
                        │  ┌─ Pipeline N ──────────────────────────────────────┐   │
                        │  │                                                    │   │
                        │  │  DataPipeline → KpiEngine → RulesEngine           │   │
                        │  │       │             │            │                  │   │
                        │  │       │             │            │                  │   │
                        │  │  (kpi trace)   (kpi trace)  (rules trace)         │   │
                        │  │       │             │            │                  │   │
                        │  │       └─────────────┼────────────┘                 │   │
                        │  │                     │                              │   │
                        │  │                     v                              │   │
                        │  │         ┌─ DiagnosticCollector ────────────┐       │   │
                        │  │         │  (pro Pipeline, POJO)            │       │   │
                        │  │         │                                  │       │   │
                        │  │         │  VerbosityLevel: OFF|MIN|STD|FULL│       │   │
                        │  │         │  Feature-Toggles per Dimension   │       │   │
                        │  │         │  RingBuffer<DiagnosticEntry>     │       │   │
                        │  │         └────────────┬────────────────────┘       │   │
                        │  │                      │                            │   │
                        │  │  DecisionArbiter → RiskGate → OMS                │   │
                        │  │       │               │          │                │   │
                        │  │  (arbiter trace)  (oms trace) (oms trace)        │   │
                        │  │       │               │          │                │   │
                        │  │       └───────────────┴──────────┘                │   │
                        │  │                      │                            │   │
                        │  │                      v                            │   │
                        │  │              DiagnosticCollector                  │   │
                        │  └──────────────────────────────────────────────────┘   │
                        │                         │                                │
                        │                         v                                │
                        │  ┌─ DiagnosticStreamService (Singleton) ──────────────┐  │
                        │  │  Aggregates across pipelines                       │  │
                        │  │  Serves SSE + REST Endpoints                       │  │
                        │  └──────────────────┬────────────────────────────────┘  │
                        │                     │                                    │
                        └─────────────────────┼────────────────────────────────────┘
                                              │
                              ┌────────────────┼──────────────────┐
                              │                │                  │
                              v                v                  v
                     SSE Stream          REST Snapshot      REST Control
              /diagnostics/stream    /diagnostics/snapshot  /diagnostics/config
                              │                │                  │
                              v                v                  v
                        ┌─────────────────────────────────────────────┐
                        │              React Frontend                  │
                        │                                              │
                        │  ┌─ Diagnostics Feature ──────────────────┐ │
                        │  │                                         │ │
                        │  │  VerbositySelector (Dropdown)           │ │
                        │  │  FeatureToggleCheckboxes                │ │
                        │  │  DiagnosticTraceFeed (SSE, 100 Entries) │ │
                        │  │  DiagnosticSnapshotPanel (REST)         │ │
                        │  └─────────────────────────────────────────┘ │
                        └──────────────────────────────────────────────┘
```

---

## Scope

### In Scope

**odin-api (DTOs, Enums):**
- `VerbosityLevel` Enum (`OFF`, `MINIMAL`, `STANDARD`, `FULL`)
- `DiagnosticFeature` Enum (6 Dimensionen: `RULES`, `ARBITER`, `LLM`, `OMS`, `KPI`, `PIPELINE`)
- `DiagnosticConfig` Record (VerbosityLevel + Set<DiagnosticFeature> + Ringbuffer-Groesse)
- `DiagnosticEntry` Record (Timestamp, Feature, barIndex, instrumentId, Payload)
- `DiagnosticSnapshot` Record (aktuelle Config + Map<Feature, List<DiagnosticEntry>>)

**odin-core (Collector, Registry):**
- `DiagnosticCollector` — pro-Pipeline-POJO mit thread-safe Ringbuffer
- `DiagnosticRegistry` — Singleton, verwaltet alle DiagnosticCollectors
- Instrumentierung der bestehenden Pipeline-Komponenten (via Collector-Injection)

**odin-app (Endpoints, SSE):**
- `DiagnosticController` — REST-Endpoints (config, snapshot, trace)
- `DiagnosticStreamController` — SSE-Endpoint fuer Echtzeit-Trace
- `DiagnosticSseEventType` — neue SSE-Event-Typen fuer den Diagnostic-Stream

**odin-frontend (Diagnostics-Panel):**
- Neues Feature `diagnostics/` in der Feature-Ordnerstruktur
- `DiagnosticsPanel` Komponente (collapsible, in Backtest-Detail und Live-Dashboard)
- `VerbositySelector` (Dropdown)
- `FeatureToggleCheckboxes` (6 Checkboxen)
- `DiagnosticTraceFeed` (SSE-basiert, max. 100 Eintraege)

### Out of Scope

- Persistierung von Trace-Daten in PostgreSQL (nur In-Memory-Ringbuffer)
- Integration mit dem bestehenden EventLog (Trace-Daten sind KEIN Teil des Audit-Trails)
- Metriken/Micrometer-Integration (kommt spaeter, separates Ticket)
- Frontend-seitige Filter/Suche ueber historische Traces (v1: nur Live-Feed)
- Performance-Profiling (CPU/Memory) — nur fachliche Decision-Traces
- Aenderungen an bestehenden SSE-Streams oder REST-Endpoints

---

## Akzeptanzkriterien

- [ ] **AC-1:** `VerbosityLevel` Enum existiert in `odin-api` mit genau 4 Werten: `OFF`, `MINIMAL`, `STANDARD`, `FULL`.
- [ ] **AC-2:** `DiagnosticFeature` Enum existiert in `odin-api` mit genau 6 Werten: `RULES`, `ARBITER`, `LLM`, `OMS`, `KPI`, `PIPELINE`.
- [ ] **AC-3:** `DiagnosticCollector` ist ein pro-Pipeline-POJO (kein Spring-Bean), das von `PipelineFactory` instanziiert wird. Es enthaelt einen thread-safe Ringbuffer mit konfigurierbarer Kapazitaet.
- [ ] **AC-4:** Im `OFF`-Modus erzeugen alle `record()`-Aufrufe auf dem `DiagnosticCollector` **zero overhead**: sofortige Rueckkehr ohne Speicherallokation, ohne Locking, ohne Seiteneffekte. Verifiziert durch Unit-Test und JMH-Benchmark-Annotation.
- [ ] **AC-5:** `POST /api/v1/diagnostics/config` akzeptiert ein JSON-Body mit `verbosityLevel` und `enabledFeatures` (Array von DiagnosticFeature) und aendert die Konfiguration zur Laufzeit fuer alle Pipelines. Aenderung ist sofort wirksam (naechster Bar-Close).
- [ ] **AC-6:** `GET /api/v1/diagnostics/snapshot` liefert den aktuellen Zustand aller aktivierten Features als JSON-Objekt. Wenn OFF: leeres Objekt `{}`.
- [ ] **AC-7:** `GET /api/v1/diagnostics/stream` liefert einen SSE-Stream mit Diagnostic-Events. Events werden pro Bar-Close gepusht, gefiltert nach aktiven Features und VerbosityLevel.
- [ ] **AC-8:** `GET /api/v1/diagnostics/trace?runId=&barIndex=&features=` liefert historische Trace-Entries aus dem Ringbuffer. Wenn barIndex oder features nicht angegeben: alle verfuegbaren Entries.
- [ ] **AC-9:** Jede der 6 Feature-Dimensionen kann einzeln ein/ausgeschaltet werden, unabhaengig vom VerbosityLevel. VerbosityLevel steuert die **Datendichte** pro Feature, Feature-Toggles steuern **welche Features** ueberhaupt Daten liefern.
- [ ] **AC-10:** Der Ringbuffer ist per-Pipeline isoliert (keine Vermischung von Instrument-Daten). Die Ringbuffer-Kapazitaet ist konfigurierbar ueber `odin.diagnostics.buffer-capacity` (Default: 500 Entries).
- [ ] **AC-11:** Frontend zeigt ein collapsible Diagnostics-Panel in der Backtest-Detail-Ansicht und im Live-Dashboard. Das Panel enthaelt: VerbositySelector (Dropdown), Feature-Checkboxes (6 Stueck), Live-Trace-Feed (SSE, max. 100 Eintraege, FIFO-Verdraengung).
- [ ] **AC-12:** Trace-Daten werden NICHT in PostgreSQL persistiert — ausschliesslich In-Memory-Ringbuffer.
- [ ] **AC-13:** Trace-Daten werden NICHT in den regulaeren EventLog (`PostgresEventLog` / `AuditEventDrainer`) geschrieben.
- [ ] **AC-14:** `RulesEngine`, `DecisionArbiter`, `KpiEngine`, `OrderManagementService` und `PipelineStateMachine` sind mit `DiagnosticCollector.record()`-Aufrufen instrumentiert. Die Instrumentierung ist non-invasiv: sie aendert kein Verhalten, keine Signaturen, keine Rueckgabewerte.
- [ ] **AC-15:** Unit-Tests fuer alle neuen Klassen. Mindestens: `DiagnosticCollectorTest`, `DiagnosticRingBufferTest`, `DiagnosticConfigTest`, `VerbosityLevelTest`.

---

## Technische Spezifikation

### 1. Verbosity-Modell

Das VerbosityLevel steuert die **Datendichte** (wie viel pro Feature geloggt wird), nicht welche Features aktiv sind (das steuern die Feature-Toggles).

| Level | Bedeutung | Beispiel: `RULES`-Feature | Beispiel: `KPI`-Feature |
|-------|-----------|--------------------------|------------------------|
| `OFF` | Zero overhead, keine Daten | Nichts | Nichts |
| `MINIMAL` | Nur kritische Entscheidungspunkte | Entry/Exit/Reject mit RejectReason | Kein KPI-Trace (KPIs sind bei MINIMAL nie kritisch genug) |
| `STANDARD` | Decision-Chain pro Bar | Alle Gate-Evaluierungen, Regime-Bestimmung, Intent-Erzeugung | RSI, ATR, EMA-Kreuzungsstatus pro Bar |
| `FULL` | Alle Zwischenwerte | Zusaetzlich: jede einzelne Bedingungspruefung mit Vergleichswert | Alle Indikatoren mit allen Timeframes, Warmup-Status, Berechnungsdauer |

### 2. Feature-Dimensionen (Detail)

#### 2.1 `RULES` — RulesEngine-Trace

| Verbosity | Inhalt |
|-----------|--------|
| MINIMAL | Entry/Exit-Intent oder Reject mit `RejectReason`, Regime |
| STANDARD | + Gate-Evaluierungen (Warmup-Guard, Regime-Confidence-Gate, Quant-Score-Threshold), Pattern-State-Machine-Zustand |
| FULL | + Einzelne Bedingungspruefungen (`EMA(9) > EMA(21)? 152.34 > 151.89 → true`), alle Pattern-State-Uebergaenge |

#### 2.2 `ARBITER` — DecisionArbiter-Trace

| Verbosity | Inhalt |
|-----------|--------|
| MINIMAL | Finale Entscheidung (ENTRY/EXIT/NO_ACTION/REJECT) mit Begruendung |
| STANDARD | + QuantScore (overall, Einzel-Scores), vetoActive, effectiveThreshold, LlmAnalysis-Metadaten (regime, confidence, freshness) |
| FULL | + Vollstaendiger Vergleich Rules-Intent vs. Quant-Score vs. LLM-Features, FusedRegime-Berechnung |

#### 2.3 `LLM` — LLM-Call-Trace

| Verbosity | Inhalt |
|-----------|--------|
| MINIMAL | Call-Timestamp, Provider, Latenz, ValidationResult (VALID/SCHEMA_ERROR/...) |
| STANDARD | + Regime, regimeConfidence, patternCandidates (Enum-Liste), Sampling-Policy-Trigger-Grund |
| FULL | + Prompt-Auszug (erste 500 Zeichen), Response-Auszug (erste 1000 Zeichen), vollstaendige LlmAnalysis-Fields |

#### 2.4 `OMS` — Order Management Trace

| Verbosity | Inhalt |
|-----------|--------|
| MINIMAL | Order-Submit, Fill, Cancel, Reject mit clientOrderId |
| STANDARD | + PositionState-Uebergaenge, Trailing-Stop-Level, Tranchen-Status, PnL-Akkumulation |
| FULL | + Repricing-Zyklen, Stop-Nachfuehrungs-Berechnungen, Tick-basierte Trail-Updates |

#### 2.5 `KPI` — Indikator-Trace

| Verbosity | Inhalt |
|-----------|--------|
| MINIMAL | Nichts (KPIs sind bei MINIMAL nie eigenstaendig relevant) |
| STANDARD | RSI, ATR, EMA(9), EMA(21), VWAP, ADX pro Bar, WarmupComplete-Flag |
| FULL | + Alle Indikatoren aller Timeframes, Bollinger-Baender, Berechnungsdauer, Volumen-Profil |

#### 2.6 `PIPELINE` — Pipeline-State-Machine-Trace

| Verbosity | Inhalt |
|-----------|--------|
| MINIMAL | State-Transitions (OBSERVING → POSITIONED) mit Trigger-Grund |
| STANDARD | + Decision-Cycle-Nummer, barIndex, MarketClock-Timestamp, Cycle-Count |
| FULL | + Bar-Close-Barrier-Timing, Decision-Cycle-Dauer (ms), DegradationMode |

### 3. Neue Klassen und Interfaces

#### 3.1 `VerbosityLevel` (odin-api)

Paket: `de.its.odin.api.model`

```java
/**
 * Controls the data density of the diagnostic layer.
 * Higher levels produce more data but incur more overhead.
 * <p>
 * OFF must produce exactly zero overhead: no allocations, no locking,
 * no method delegation. All trace calls short-circuit immediately.
 *
 * @see DiagnosticFeature
 * @see docs/backend/architecture/00-system-overview.md
 */
public enum VerbosityLevel {

    /** No diagnostic data collected. Zero overhead. */
    OFF,

    /** Only critical decision points (Entry/Exit/Reject). */
    MINIMAL,

    /** Decision chain per bar plus KPI snapshot. Normal diagnosis. */
    STANDARD,

    /** All intermediate values, all gate states, all LLM I/O. Deep diagnosis. */
    FULL
}
```

#### 3.2 `DiagnosticFeature` (odin-api)

Paket: `de.its.odin.api.model`

```java
/**
 * Enumerates the independently toggleable dimensions of the diagnostic layer.
 * Each feature can be enabled or disabled regardless of the VerbosityLevel.
 * VerbosityLevel controls data density; DiagnosticFeature controls scope.
 *
 * @see VerbosityLevel
 */
public enum DiagnosticFeature {

    /** RulesEngine trace: gate states, entry/exit conditions, regime determination. */
    RULES,

    /** DecisionArbiter trace: quant vote, LLM vote, fused regime, block reason. */
    ARBITER,

    /** LLM call trace: prompt excerpt, response, sampling decision, latency. */
    LLM,

    /** OMS trace: orders, fills, position state, PnL accumulation, trailing stops. */
    OMS,

    /** KPI values per bar: all indicator values from the KPI engine. */
    KPI,

    /** Pipeline state machine transitions: state changes, triggers, cycle counts. */
    PIPELINE
}
```

#### 3.3 `DiagnosticConfig` (odin-api)

Paket: `de.its.odin.api.dto`

```java
/**
 * Runtime configuration for the diagnostic layer.
 * Immutable record — changes produce a new instance.
 *
 * @param verbosityLevel the global data density level
 * @param enabledFeatures set of features that produce trace data
 *                        (empty set = no features active, equivalent to OFF behavior)
 * @param bufferCapacity  maximum entries per pipeline ringbuffer
 */
public record DiagnosticConfig(
        VerbosityLevel verbosityLevel,
        Set<DiagnosticFeature> enabledFeatures,
        int bufferCapacity
) {
    /** Default configuration: OFF, no features, 500 entries buffer. */
    public static final DiagnosticConfig DEFAULT = new DiagnosticConfig(
            VerbosityLevel.OFF,
            Set.of(),
            500
    );

    /**
     * Returns true if the given feature is currently active
     * (verbosity is not OFF AND feature is in the enabled set).
     */
    public boolean isActive(DiagnosticFeature feature) {
        return verbosityLevel != VerbosityLevel.OFF
                && enabledFeatures.contains(feature);
    }
}
```

#### 3.4 `DiagnosticEntry` (odin-api)

Paket: `de.its.odin.api.dto`

```java
/**
 * A single diagnostic trace entry produced by an instrumented component.
 *
 * @param timestamp    MarketClock timestamp when the entry was recorded
 * @param feature      which diagnostic dimension produced this entry
 * @param instrumentId the pipeline instrument identifier
 * @param barIndex     the bar index within the current run (0-based)
 * @param verbosity    the verbosity level at which this entry was produced
 * @param label        short human-readable label (e.g. "RSI_CHECK", "ENTRY_REJECT")
 * @param payload      structured data as Map — content varies by feature and verbosity
 */
public record DiagnosticEntry(
        Instant timestamp,
        DiagnosticFeature feature,
        String instrumentId,
        long barIndex,
        VerbosityLevel verbosity,
        String label,
        Map<String, Object> payload
) {}
```

#### 3.5 `DiagnosticSnapshot` (odin-api)

Paket: `de.its.odin.api.dto`

```java
/**
 * Point-in-time snapshot of all diagnostic data across all pipelines.
 * Returned by GET /api/v1/diagnostics/snapshot.
 *
 * @param config          current diagnostic configuration
 * @param pipelines       map of instrumentId to list of recent entries
 * @param snapshotTime    MarketClock timestamp when the snapshot was taken
 * @param totalEntries    total number of entries across all pipelines
 */
public record DiagnosticSnapshot(
        DiagnosticConfig config,
        Map<String, List<DiagnosticEntry>> pipelines,
        Instant snapshotTime,
        long totalEntries
) {}
```

#### 3.6 `DiagnosticRingBuffer` (odin-core)

Paket: `de.its.odin.core.diagnostics`

```java
/**
 * Fixed-capacity, thread-safe ring buffer for diagnostic entries.
 * Per-pipeline isolation: each pipeline gets its own buffer instance.
 * <p>
 * Thread-safety: synchronized on all mutating operations. Read operations
 * return defensive copies. This is acceptable because diagnostic data
 * is not on the critical trading path.
 *
 * @see DiagnosticCollector
 */
public class DiagnosticRingBuffer {

    private final DiagnosticEntry[] buffer;
    private final int capacity;
    private long writeIndex;
    private long totalEntries;

    /**
     * Creates a new DiagnosticRingBuffer.
     *
     * @param capacity maximum number of entries to retain (must be positive)
     * @throws IllegalArgumentException if capacity is not positive
     */
    public DiagnosticRingBuffer(int capacity) { ... }

    /**
     * Adds an entry to the buffer. Oldest entry is evicted if full.
     *
     * @param entry the diagnostic entry to add
     */
    public synchronized void add(DiagnosticEntry entry) { ... }

    /**
     * Returns all entries currently in the buffer, ordered oldest-first.
     *
     * @return defensive copy of buffered entries
     */
    public synchronized List<DiagnosticEntry> getAll() { ... }

    /**
     * Returns entries matching the given bar index, ordered oldest-first.
     *
     * @param barIndex the bar index to filter by
     * @return matching entries (defensive copy)
     */
    public synchronized List<DiagnosticEntry> getByBarIndex(long barIndex) { ... }

    /**
     * Returns entries matching any of the given features, ordered oldest-first.
     *
     * @param features the features to filter by
     * @return matching entries (defensive copy)
     */
    public synchronized List<DiagnosticEntry> getByFeatures(Set<DiagnosticFeature> features) { ... }

    /**
     * Returns the total number of entries ever written (including evicted).
     *
     * @return total entry count
     */
    public synchronized long getTotalEntries() { ... }

    /** Clears all entries from the buffer. */
    public synchronized void clear() { ... }
}
```

#### 3.7 `DiagnosticCollector` (odin-core)

Paket: `de.its.odin.core.diagnostics`

```java
/**
 * Per-pipeline diagnostic data collector. POJO — instantiated by PipelineFactory,
 * NOT a Spring bean. Holds a reference to the shared DiagnosticConfig (AtomicReference)
 * and a per-pipeline DiagnosticRingBuffer.
 * <p>
 * The record() method is the primary instrumentation point. It is called by
 * pipeline components (RulesEngine, DecisionArbiter, KpiEngine, OMS, PipelineStateMachine)
 * to emit trace data. The method short-circuits immediately if:
 * <ol>
 *   <li>VerbosityLevel is OFF</li>
 *   <li>The requested feature is not in the enabled set</li>
 *   <li>The requested verbosity exceeds the configured level</li>
 * </ol>
 * <p>
 * Zero-overhead guarantee in OFF mode: the method checks a single volatile
 * VerbosityLevel field and returns. No allocation, no locking, no delegation.
 *
 * @see DiagnosticRegistry
 * @see docs/backend/architecture/00-system-overview.md
 */
public class DiagnosticCollector {

    private final String instrumentId;
    private final AtomicReference<DiagnosticConfig> configRef;
    private final DiagnosticRingBuffer ringBuffer;
    private final MarketClock marketClock;

    /**
     * Creates a new DiagnosticCollector for a specific pipeline.
     *
     * @param instrumentId the pipeline instrument identifier
     * @param configRef    shared configuration reference (updated at runtime via REST)
     * @param marketClock  time source for entry timestamps
     * @param bufferCapacity ringbuffer capacity
     */
    public DiagnosticCollector(
            String instrumentId,
            AtomicReference<DiagnosticConfig> configRef,
            MarketClock marketClock,
            int bufferCapacity
    ) { ... }

    /**
     * Records a diagnostic entry if the current configuration permits it.
     * <p>
     * Short-circuits immediately (zero overhead) if:
     * - VerbosityLevel is OFF
     * - Feature is not enabled
     * - requiredVerbosity exceeds current VerbosityLevel
     *
     * @param feature           the diagnostic dimension
     * @param requiredVerbosity minimum verbosity required for this entry
     * @param barIndex          current bar index in the run
     * @param label             short human-readable label
     * @param payloadSupplier   lazy supplier for the payload map (NOT called if short-circuited)
     */
    public void record(
            DiagnosticFeature feature,
            VerbosityLevel requiredVerbosity,
            long barIndex,
            String label,
            Supplier<Map<String, Object>> payloadSupplier
    ) { ... }

    /**
     * Returns all entries currently in the ring buffer.
     *
     * @return list of entries, oldest-first
     */
    public List<DiagnosticEntry> getEntries() { ... }

    /**
     * Returns entries for a specific bar index.
     *
     * @param barIndex the bar index to query
     * @return matching entries
     */
    public List<DiagnosticEntry> getEntriesByBarIndex(long barIndex) { ... }

    /**
     * Returns entries for specific features.
     *
     * @param features the features to filter by
     * @return matching entries
     */
    public List<DiagnosticEntry> getEntriesByFeatures(Set<DiagnosticFeature> features) { ... }

    /**
     * Returns the instrument ID this collector is associated with.
     *
     * @return instrument identifier
     */
    public String getInstrumentId() { ... }
}
```

**Kritisches Design-Detail: `payloadSupplier`**

Der `Supplier<Map<String, Object>>` wird NUR aufgerufen, wenn die Pruefung ergibt, dass der Entry tatsaechlich aufgezeichnet wird. Im OFF-Modus oder bei deaktiviertem Feature wird der Supplier nie evaluiert — kein Map-Aufbau, keine String-Operationen, keine Allokation. Das ist der Schluessel zur Zero-Overhead-Garantie.

```java
// Aufrufbeispiel in RulesEngine:
diagnosticCollector.record(
    DiagnosticFeature.RULES,
    VerbosityLevel.STANDARD,
    barIndex,
    "REGIME_DETERMINATION",
    () -> Map.of(
        "kpiRegime", kpiRegime.name(),
        "llmRegime", llmRegime != null ? llmRegime.name() : "NONE",
        "fusedRegime", fusedRegime.name(),
        "adx", String.valueOf(adxValue),
        "emaShort", String.valueOf(emaShortValue),
        "emaLong", String.valueOf(emaLongValue)
    )
);
```

#### 3.8 `DiagnosticRegistry` (odin-core)

Paket: `de.its.odin.core.diagnostics`

```java
/**
 * Singleton registry that manages DiagnosticCollectors across all pipelines.
 * Spring-managed bean in odin-core. Holds the shared DiagnosticConfig
 * (AtomicReference) that all collectors read from.
 * <p>
 * The registry is the single point of control for diagnostic configuration changes
 * (via REST endpoint). Configuration changes are immediately visible to all collectors
 * through the shared AtomicReference — no restart, no pipeline re-creation needed.
 *
 * @see DiagnosticCollector
 */
@Component
public class DiagnosticRegistry {

    private final AtomicReference<DiagnosticConfig> configRef;
    private final ConcurrentHashMap<String, DiagnosticCollector> collectors;

    /** Creates a new DiagnosticRegistry with default configuration. */
    public DiagnosticRegistry() { ... }

    /**
     * Creates or retrieves a DiagnosticCollector for the given instrument.
     * Called by PipelineFactory during pipeline creation.
     *
     * @param instrumentId the pipeline instrument identifier
     * @param marketClock  time source
     * @return the collector for this instrument
     */
    public DiagnosticCollector getOrCreateCollector(String instrumentId, MarketClock marketClock) { ... }

    /**
     * Updates the diagnostic configuration. Changes are immediately visible
     * to all existing collectors via the shared AtomicReference.
     *
     * @param newConfig the new configuration
     */
    public void updateConfig(DiagnosticConfig newConfig) { ... }

    /** Returns the current diagnostic configuration. */
    public DiagnosticConfig getConfig() { ... }

    /**
     * Returns a snapshot of all diagnostic data across all pipelines.
     *
     * @return the current diagnostic snapshot
     */
    public DiagnosticSnapshot buildSnapshot() { ... }

    /**
     * Returns the collector for a specific instrument, or empty if not registered.
     *
     * @param instrumentId the instrument to look up
     * @return optional collector
     */
    public Optional<DiagnosticCollector> getCollector(String instrumentId) { ... }

    /** Removes the collector for the given instrument. Called during pipeline teardown. */
    public void removeCollector(String instrumentId) { ... }
}
```

#### 3.9 `PipelineFactory` Erweiterung (odin-core)

`PipelineFactory` erhaelt `DiagnosticRegistry` als neuen Konstruktor-Parameter. Bei Pipeline-Erstellung wird ein `DiagnosticCollector` ueber die Registry erzeugt und an alle instrumentierten Komponenten uebergeben:

```java
// In PipelineFactory.createPipeline():
DiagnosticCollector diagnosticCollector = diagnosticRegistry.getOrCreateCollector(
    instrumentId, marketClock);

// Injection in pipeline components:
KpiEngine kpiEngine = new KpiEngine(..., diagnosticCollector);
RulesEngine rulesEngine = new RulesEngine(..., diagnosticCollector);
DecisionArbiter arbiter = new DecisionArbiter(..., diagnosticCollector);
OrderManagementService oms = new OrderManagementService(..., diagnosticCollector);
PipelineStateMachine fsm = new PipelineStateMachine(..., diagnosticCollector);
```

**Abwaertskompatibilitaet:** Bestehende Konstruktoren der Komponenten werden NICHT geaendert. Stattdessen erhalten die Klassen einen neuen Konstruktor-Parameter `DiagnosticCollector`. Da diese Klassen POJOs sind (keine Spring-Beans), ist die PipelineFactory der einzige Aufrufer — es gibt keine Breaking Changes.

### 4. API-Spezifikation

#### 4.1 `POST /api/v1/diagnostics/config` — Konfiguration aendern

**Request:**
```json
{
    "verbosityLevel": "STANDARD",
    "enabledFeatures": ["RULES", "ARBITER", "KPI"]
}
```

**Response (200 OK):**
```json
{
    "verbosityLevel": "STANDARD",
    "enabledFeatures": ["RULES", "ARBITER", "KPI"],
    "bufferCapacity": 500,
    "appliedAt": "2026-02-25T14:30:00.123Z"
}
```

**Response (400 Bad Request):**
```json
{
    "error": "INVALID_VERBOSITY_LEVEL",
    "message": "Unknown verbosity level: TURBO. Valid values: OFF, MINIMAL, STANDARD, FULL"
}
```

**Semantik:**
- Aenderung ist sofort wirksam (naechster `record()`-Aufruf sieht neue Config)
- Sendet `enabledFeatures` als leeres Array um alle Features zu deaktivieren
- `bufferCapacity` ist NICHT per REST aenderbar (nur via Properties) — die Response zeigt den aktuellen Wert
- Feature-Toggles und VerbosityLevel sind orthogonal: `STANDARD` + `[]` = keine Daten (keine Features aktiv). `OFF` + `[RULES, KPI]` = keine Daten (OFF uebersteuert alles)

#### 4.2 `GET /api/v1/diagnostics/snapshot` — Aktueller Zustand

**Response (200 OK):**
```json
{
    "config": {
        "verbosityLevel": "STANDARD",
        "enabledFeatures": ["RULES", "ARBITER"],
        "bufferCapacity": 500
    },
    "pipelines": {
        "AAPL": [
            {
                "timestamp": "2026-02-25T14:30:00.123Z",
                "feature": "RULES",
                "instrumentId": "AAPL",
                "barIndex": 142,
                "verbosity": "STANDARD",
                "label": "ENTRY_REJECT",
                "payload": {
                    "reason": "REGIME_UNCERTAIN",
                    "regime": "UNCERTAIN",
                    "regimeConfidence": 0.32
                }
            }
        ],
        "MSFT": []
    },
    "snapshotTime": "2026-02-25T14:30:01.456Z",
    "totalEntries": 287
}
```

**Semantik:**
- Wenn VerbosityLevel = OFF: `pipelines` ist leeres Objekt `{}`
- Entries sind nach Timestamp absteigend sortiert (neueste zuerst)
- Response-Groesse ist durch die Ringbuffer-Kapazitaet natuerlich begrenzt

#### 4.3 `GET /api/v1/diagnostics/stream` — Echtzeit-SSE-Trace

**SSE Event Format:**
```
event: diagnostic-trace
id: 1523
data: {"timestamp":"2026-02-25T14:30:00.123Z","feature":"RULES","instrumentId":"AAPL","barIndex":142,"verbosity":"STANDARD","label":"REGIME_DETERMINATION","payload":{"kpiRegime":"TREND_UP","llmRegime":"TREND_UP","fusedRegime":"TREND_UP"}}

event: diagnostic-trace
id: 1524
data: {"timestamp":"2026-02-25T14:30:00.124Z","feature":"ARBITER","instrumentId":"AAPL","barIndex":142,"verbosity":"STANDARD","label":"DECISION","payload":{"intent":"ENTRY","quantScore":0.72,"vetoActive":false}}
```

**Query-Parameter:**
| Parameter | Typ | Default | Beschreibung |
|-----------|-----|---------|--------------|
| `instrumentId` | String | alle | Filter auf ein Instrument |
| `features` | String (Komma-separiert) | alle aktiven | Filter auf Features |

**Semantik:**
- Stream liefert Events fuer alle aktiven Pipelines (oder gefiltert per Query-Param)
- Reconnect via `Last-Event-ID` wird unterstuetzt (analog bestehender SSE-Streams)
- Heartbeat alle 5s (wiederverwendet bestehenden Heartbeat-Mechanismus)
- Stream liefert keine Events wenn VerbosityLevel = OFF

#### 4.4 `GET /api/v1/diagnostics/trace` — Historischer Trace

**Query-Parameter:**
| Parameter | Typ | Pflicht | Beschreibung |
|-----------|-----|---------|--------------|
| `runId` | String | Nein | Filter auf einen Run (Default: aktueller Run) |
| `instrumentId` | String | Nein | Filter auf ein Instrument |
| `barIndex` | Long | Nein | Filter auf einen bestimmten Bar |
| `features` | String (Komma-separiert) | Nein | Filter auf Features |
| `limit` | Integer | Nein (Default: 100) | Maximale Anzahl Entries |

**Response (200 OK):**
```json
{
    "entries": [
        {
            "timestamp": "2026-02-25T14:30:00.123Z",
            "feature": "RULES",
            "instrumentId": "AAPL",
            "barIndex": 142,
            "verbosity": "STANDARD",
            "label": "REGIME_DETERMINATION",
            "payload": { ... }
        }
    ],
    "totalAvailable": 287,
    "filtered": true
}
```

**Semantik:**
- Liest aus dem In-Memory-Ringbuffer (nicht aus DB)
- Wenn kein runId angegeben: aktueller aktiver Run
- Wenn Ringbuffer leer oder Diagnostics OFF: leere `entries`-Liste

### 5. Instrumentierungspunkte (bestehende Klassen)

Die folgenden bestehenden Klassen erhalten `DiagnosticCollector` als neuen Konstruktor-Parameter und werden mit `record()`-Aufrufen instrumentiert. Keine bestehenden Signaturen werden geaendert — nur neue Parameter hinzugefuegt.

| Klasse | Modul | Feature-Dimension | Instrumentierungspunkte |
|--------|-------|-------------------|------------------------|
| `RulesEngine` | odin-brain | `RULES` | Regime-Bestimmung, Gate-Evaluierungen (Warmup, Confidence, Quant-Threshold), Entry/Exit-Bedingungspruefungen, Pattern-State-Machine-Uebergaenge |
| `EntryRules` | odin-brain | `RULES` | Einzelne Entry-Bedingungen pro Regime (EMA-Vergleich, RSI-Check, VWAP-Offset) |
| `ExitRules` | odin-brain | `RULES` | Exit-Trigger-Evaluierung (Prioritaetsordnung Prio 1-7) |
| `DecisionArbiter` | odin-brain | `ARBITER` | Intent-vs-Score-Vergleich, Veto-Pruefung, finale Entscheidung |
| `KpiEngine` | odin-brain | `KPI` | Berechnete Indikatorwerte pro Bar, Warmup-Status |
| `OrderManagementService` | odin-execution | `OMS` | Order-Submit, Fill-Handling, Trailing-Stop-Updates, Tranchen-Status, PnL-Berechnung |
| `RiskGate` | odin-execution | `OMS` | Position-Sizing-Ergebnis, Limit-Pruefungen, Reject-Gruende |
| `PipelineStateMachine` | odin-core | `PIPELINE` | State-Transitions, Trigger-Gruende, Cycle-Counter |
| `LlmAnalystOrchestrator` | odin-brain | `LLM` | Call-Timing, Sampling-Policy-Decision, Response-Validierung |
| `TradingPipeline` | odin-core | `PIPELINE` | Decision-Cycle-Start/Ende, Bar-Index, DegradationMode |

**Instrumentierungs-Pattern (fuer jede Klasse identisch):**

```java
// Neuer Konstruktor-Parameter:
private final DiagnosticCollector diagnosticCollector;

// Instrumentierung (Beispiel in RulesEngine):
public RulesResult evaluate(MarketSnapshot snapshot, IndicatorResult indicators,
                             LlmAnalysis llmAnalysis, PipelineState state, long barIndex) {
    // ... bestehende Logik ...

    Regime fusedRegime = determineRegime(indicators, llmAnalysis);

    diagnosticCollector.record(
        DiagnosticFeature.RULES,
        VerbosityLevel.STANDARD,
        barIndex,
        "REGIME_DETERMINATION",
        () -> Map.of(
            "kpiRegime", kpiRegime.name(),
            "llmRegime", llmAnalysis != null ? llmAnalysis.regime().name() : "ABSENT",
            "fusedRegime", fusedRegime.name(),
            "regimeConfidence", llmAnalysis != null
                ? String.valueOf(llmAnalysis.regimeConfidence()) : "N/A"
        )
    );

    // ... weiter mit bestehender Logik ...
}
```

### 6. Konfiguration (Backend)

```properties
# odin-core.properties (Diagnostics-Abschnitt)

# Default verbosity level at startup
odin.diagnostics.default-verbosity=OFF

# Default enabled features at startup (comma-separated, empty = none)
odin.diagnostics.default-features=

# Ring buffer capacity per pipeline (number of entries)
odin.diagnostics.buffer-capacity=500

# SSE stream: maximum entries to send on reconnect replay
odin.diagnostics.sse.replay-buffer-size=100

# SSE stream: heartbeat interval (reuses global SSE heartbeat)
# odin.frontend.sse.heartbeat-interval-ms=5000 (existing)
```

Konfigurationsstruktur (Record-based, `@ConfigurationProperties`):

```java
/**
 * Configuration properties for the diagnostic layer.
 *
 * @param defaultVerbosity default verbosity level at startup
 * @param defaultFeatures  comma-separated list of default-enabled features
 * @param bufferCapacity   ring buffer capacity per pipeline
 * @param sse              SSE-specific diagnostic config
 */
@ConfigurationProperties(prefix = "odin.diagnostics")
@Validated
public record DiagnosticsProperties(
        @NotNull VerbosityLevel defaultVerbosity,
        String defaultFeatures,
        @Min(10) @Max(10000) int bufferCapacity,
        @NotNull @Valid SseConfig sse
) {
    /**
     * SSE-specific diagnostic configuration.
     *
     * @param replayBufferSize maximum entries to replay on reconnect
     */
    public record SseConfig(
            @Min(10) @Max(1000) int replayBufferSize
    ) {}
}
```

### 7. Frontend-Spezifikation

#### 7.1 Feature-Ordnerstruktur

```
src/domains/diagnostics/
├── components/
│   ├── DiagnosticsPanel.tsx           # Collapsible container
│   ├── VerbositySelector.tsx          # Dropdown: OFF|MINIMAL|STANDARD|FULL
│   ├── FeatureToggleCheckboxes.tsx    # 6 checkboxes, one per DiagnosticFeature
│   ├── DiagnosticTraceFeed.tsx        # SSE live feed, max 100 entries
│   └── DiagnosticEntryRow.tsx         # Single trace entry rendering
├── hooks/
│   ├── useDiagnosticConfig.ts         # GET/POST /api/v1/diagnostics/config
│   ├── useDiagnosticStream.ts         # SSE /api/v1/diagnostics/stream
│   └── useDiagnosticSnapshot.ts       # GET /api/v1/diagnostics/snapshot
├── types/
│   └── diagnostics.ts                 # TypeScript types matching backend DTOs
└── api/
    └── diagnosticsApi.ts              # REST client functions
```

#### 7.2 TypeScript Types

```typescript
// types/diagnostics.ts

type VerbosityLevel = 'OFF' | 'MINIMAL' | 'STANDARD' | 'FULL';

type DiagnosticFeature = 'RULES' | 'ARBITER' | 'LLM' | 'OMS' | 'KPI' | 'PIPELINE';

interface DiagnosticConfig {
    readonly verbosityLevel: VerbosityLevel;
    readonly enabledFeatures: DiagnosticFeature[];
    readonly bufferCapacity: number;
}

interface DiagnosticEntry {
    readonly timestamp: string;
    readonly feature: DiagnosticFeature;
    readonly instrumentId: string;
    readonly barIndex: number;
    readonly verbosity: VerbosityLevel;
    readonly label: string;
    readonly payload: Record<string, unknown>;
}

interface DiagnosticSnapshot {
    readonly config: DiagnosticConfig;
    readonly pipelines: Record<string, DiagnosticEntry[]>;
    readonly snapshotTime: string;
    readonly totalEntries: number;
}

interface DiagnosticConfigUpdateRequest {
    readonly verbosityLevel: VerbosityLevel;
    readonly enabledFeatures: DiagnosticFeature[];
}

interface DiagnosticConfigUpdateResponse extends DiagnosticConfig {
    readonly appliedAt: string;
}
```

#### 7.3 Komponenten-Verhalten

**`DiagnosticsPanel`:**
- Collapsible (collapsed by default)
- Position: unterhalb des Chart-Bereichs in Backtest-Detail und Live-Dashboard
- Header zeigt aktuelles VerbosityLevel und Anzahl aktiver Features
- `data-testid="diagnostics-panel"` fuer E2E-Tests

**`VerbositySelector`:**
- HTML `<select>` mit 4 Optionen
- `onChange` → `POST /api/v1/diagnostics/config` mit aktuellem Feature-Set
- Optimistic UI Update (sofort anzeigen, bei Fehler zurueckrollen)
- `data-testid="verbosity-selector"`

**`FeatureToggleCheckboxes`:**
- 6 Checkboxen in einer Reihe (Icons + Labels: Rules, Arbiter, LLM, OMS, KPI, Pipeline)
- Jede Checkbox-Aenderung → `POST /api/v1/diagnostics/config` mit neuem Feature-Set
- Disabled wenn VerbosityLevel = OFF (visuelle Klarheit)
- `data-testid="feature-toggle-{feature}"` (z.B. `feature-toggle-RULES`)

**`DiagnosticTraceFeed`:**
- Scrollbarer Container, neueste Entries oben
- Max. 100 Entries angezeigt, aelteste werden verdraengt (FIFO)
- Farbcodierung nach Feature (RULES=blau, ARBITER=gruen, LLM=violett, OMS=orange, KPI=grau, PIPELINE=cyan)
- Auto-Scroll nach unten bei neuen Entries (mit Scroll-Lock wenn User manuell scrollt)
- Jeder Entry zeigt: Timestamp, Feature-Badge, Label, und ein expandable Payload-JSON
- `data-testid="diagnostic-trace-feed"`

### 8. Abhaengigkeitsdiagramm (Modulgrenzen)

```
odin-api
  ├── VerbosityLevel (Enum)
  ├── DiagnosticFeature (Enum)
  ├── DiagnosticConfig (Record)
  ├── DiagnosticEntry (Record)
  └── DiagnosticSnapshot (Record)
       │
       │ (odin-core haengt von odin-api ab)
       v
odin-core
  ├── DiagnosticRingBuffer
  ├── DiagnosticCollector (Pro-Pipeline-POJO)
  └── DiagnosticRegistry (Singleton, @Component)
       │
       │ (odin-app haengt von odin-core ab)
       v
odin-app
  ├── DiagnosticController (REST)
  ├── DiagnosticStreamController (SSE)
  └── dto/DiagnosticConfigRequest
       │
       │ (Frontend konsumiert odin-app REST/SSE)
       v
odin-frontend
  └── domains/diagnostics/ (React-Komponenten)
```

**Abhaengigkeitsregeln beachtet:**
- `VerbosityLevel`, `DiagnosticFeature`, `DiagnosticConfig`, `DiagnosticEntry`, `DiagnosticSnapshot` leben in **odin-api** (keine Abhaengigkeiten)
- `DiagnosticCollector` und `DiagnosticRingBuffer` leben in **odin-core** (haengt von odin-api ab)
- `DiagnosticRegistry` ist ein **Singleton** in odin-core (Spring-Bean, da es pipeline-uebergreifend arbeitet)
- `DiagnosticCollector` ist ein **Pro-Pipeline-POJO** (keine Spring-Annotation, manuell von PipelineFactory erzeugt)
- Instrumentierte Klassen in odin-brain und odin-execution erhalten den `DiagnosticCollector` als Konstruktor-Parameter — KEIN Import von odin-core noetig, da `DiagnosticCollector` **von odin-core in die Pipeline injiziert** wird und die odin-brain/odin-execution-Klassen ihn nur als POJO-Parameter empfangen

**Wichtig: Fachmodul-Querverweise vermeiden**

`DiagnosticCollector` lebt in odin-core, aber odin-brain und odin-execution duerfen NICHT von odin-core abhaengen (Abhaengigkeitsrichtung: api <- brain, execution <- core). Loesung:

1. **Interface** `DiagnosticSink` in **odin-api** definieren (minimales Interface mit `record()`-Methode)
2. **Implementierung** `DiagnosticCollector` in **odin-core** implementiert `DiagnosticSink`
3. **Fachmodule** (brain, execution) programmieren gegen `DiagnosticSink` aus odin-api
4. **PipelineFactory** uebergibt den konkreten `DiagnosticCollector` als `DiagnosticSink`

```java
// In odin-api:
public interface DiagnosticSink {
    void record(DiagnosticFeature feature, VerbosityLevel requiredVerbosity,
                long barIndex, String label, Supplier<Map<String, Object>> payloadSupplier);
}

// No-Op-Implementierung in odin-api fuer Tests ohne Diagnostics:
public final class NoOpDiagnosticSink implements DiagnosticSink {
    public static final NoOpDiagnosticSink INSTANCE = new NoOpDiagnosticSink();
    private NoOpDiagnosticSink() {}
    @Override
    public void record(DiagnosticFeature feature, VerbosityLevel requiredVerbosity,
                       long barIndex, String label, Supplier<Map<String, Object>> payloadSupplier) {
        // Intentionally empty — zero overhead
    }
}
```

---

## Performance-Anforderungen

| Aspekt | Anforderung | Messmethode |
|--------|-------------|-------------|
| `record()` im OFF-Modus | < 5 ns pro Aufruf (Zero Overhead) | JMH-Benchmark im Unit-Test |
| `record()` im FULL-Modus | < 1 ms pro Aufruf inkl. Payload-Erzeugung | JMH-Benchmark |
| Ringbuffer `add()` | < 100 us (synchronized, aber kein Contention da single-threaded pro Pipeline) | Unit-Test mit Timing |
| SSE Diagnostic Stream | Kein Einfluss auf bestehende Monitoring-SSE-Streams | Manuell verifizieren |
| Speicherverbrauch pro Pipeline | Max. `bufferCapacity * avg_entry_size` — bei 500 Entries ca. 200 KB | Heap-Dump-Analyse |
| Decision-Cycle-Latenz | < 1% Verlaengerung bei STANDARD, < 5% bei FULL | Benchmark im Backtest |

**Kritisch:** Im Produktivbetrieb (OFF-Modus) darf der Diagnostic Layer die E2E-Latenz des Decision-Cycles (< 100ms p99, Kap 0 §10) NICHT messbar beeinflussen.

---

## Out of Scope (explizit)

- **Keine Persistierung:** Trace-Daten werden NICHT in PostgreSQL gespeichert. Ringbuffer ist In-Memory und geht bei Restart verloren. Wenn Persistierung benoetigt wird, ist das eine separate Story.
- **Kein EventLog-Integration:** Trace-Daten fliessen NICHT in den regulaeren EventLog (`PostgresEventLog`). Der EventLog bleibt fuer finale Entscheidungen, nicht fuer Zwischenwerte.
- **Kein Micrometer/JMX:** Keine Integration mit dem Observability-Stack (Kap 0 §10). Das ist eine separate Story.
- **Keine historische Analyse:** Keine Moeglichkeit, Trace-Daten vergangener Tage abzurufen. Der Ringbuffer haelt nur den aktuellen Lauf.
- **Kein Frontend-Filter:** Keine client-seitige Suche/Filter ueber Trace-Daten. V1 zeigt nur den Live-Feed.
- **Kein Security-Layer:** Die Diagnostic-Endpoints nutzen dieselbe Authentifizierung wie die bestehenden REST-Endpoints (HTTP Basic Auth, Kap 9 §3). Kein separates Rollenmodell.
- **Keine Aenderungen am bestehenden `window.__odinDiagnostics`:** Das E2E-Diagnostics-System (ODIN-051) bleibt unberuehrt. Der neue Diagnostic Layer ist ein separates System mit einem anderen Zweck (Trading-Logik-Traces vs. E2E-Test-Infrastruktur).

---

## Abhaengigkeiten

### Benoetigt von dieser Story

| Abhaengigkeit | Status | Auswirkung |
|--------------|--------|------------|
| `PipelineFactory` (odin-core) | Existiert | Erweiterung um DiagnosticRegistry-Parameter |
| `SseEmitterManager` (odin-app) | Existiert | Wiederverwendung fuer Diagnostic SSE Stream |
| `SseRingBuffer` (odin-app) | Existiert | Wiederverwendung oder Neuerstellung fuer Diagnostic Stream |
| `RulesEngine`, `DecisionArbiter`, `KpiEngine`, `OMS`, `PipelineStateMachine` | Existieren | Instrumentierung (nicht-invasive Erweiterung) |
| Bestehende SSE-Infrastruktur | Existiert | Heartbeat, Emitter-Management, Reconnect-Replay |

### Blockiert durch diese Story

Keine. Diese Story ist unabhaengig und blockiert keine anderen Stories.

### Potentiell benoetigt von zukuenftigen Stories

- Agent-gesteuertes Parameter-Tuning (Agent liest Traces und passt Config an)
- Automatisierte Strategie-Evaluation (Agent diagnostiziert Backtest-Ergebnisse)
- LLM-gesteuerte Selbstoptimierung (Agent analysiert Decision-Chains)

---

## Konzept-Referenzen

| Dokument | Abschnitt |
|----------|-----------|
| `docs/architecture/00-system-overview.md` | §3.1 Simulationsfaehigkeit, §6 Thread-Modell, §10 Observability |
| `docs/backend/architecture/06-rules-engine.md` | §3 Decision Loop, §4 Regime-Bestimmung, §8 Decision Arbiter |
| `docs/frontend/architecture/09-frontend.md` | §3 SSE/REST Kommunikation, §4 Komponenten |
| `docs/backend/guardrails/development-process.md` | §5 Implementierungsregeln, §7 Review-Gates |
| `docs/backend/architecture/01-module-architecture.md` | Abhaengigkeitsrichtung, Singleton vs. Pro-Pipeline |

---

## Guardrail-Referenzen

| Guardrail | Pfad |
|-----------|------|
| Entwicklungsprozess | `docs/backend/guardrails/development-process.md` |
| Modulstruktur | `docs/backend/guardrails/module-structure.md` |
| Frontend | `docs/frontend/guardrails/frontend.md` |
| CLAUDE.md | `T:/codebase/its_odin/CLAUDE.md` |

---

## Definition of Done

### DoD 1: Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien AC-1 bis AC-15
- [ ] `mvn compile` fehlerfrei (Gesamtprojekt, da odin-api geaendert wird)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (`DiagnosticConfig`, `DiagnosticEntry`, `DiagnosticSnapshot`)
- [ ] JavaDoc auf allen neuen/geaenderten public Klassen und Methoden
- [ ] Keine TODO/FIXME-Kommentare im abgelieferten Code
- [ ] `DiagnosticCollector` und `DiagnosticRingBuffer` sind Pro-Pipeline-POJOs ohne Spring-Annotations
- [ ] `DiagnosticRegistry` ist ein `@Component` Singleton
- [ ] `DiagnosticSink` Interface in odin-api — keine Fachmodul-Querverweise
- [ ] `NoOpDiagnosticSink.INSTANCE` fuer Tests und als Fallback
- [ ] `MarketClock.now()` fuer Timestamps in DiagnosticEntry, kein `Instant.now()`
- [ ] Port-Abstraktion: DiagnosticSink als Interface in odin-api

### DoD 2: Tests — Unit-Tests

- [ ] `VerbosityLevelTest`: Enum-Ordnung, `OFF` ist erster Wert
- [ ] `DiagnosticFeatureTest`: Alle 6 Werte vorhanden
- [ ] `DiagnosticConfigTest`: `isActive()` korrekt fuer alle Kombinationen (OFF+Feature, STANDARD+Feature, STANDARD+leeres Set)
- [ ] `DiagnosticRingBufferTest`: Add, eviction bei Ueberlauf, getAll Sortierung, getByBarIndex, getByFeatures, clear, Thread-Safety (parallele Adds)
- [ ] `DiagnosticCollectorTest`: Zero-overhead im OFF-Modus (Supplier wird NICHT aufgerufen), korrekte Filterung nach Feature und Verbosity, Config-Update ueber AtomicReference
- [ ] `DiagnosticRegistryTest`: getOrCreateCollector (Idempotenz), updateConfig propagiert zu allen Collectors, buildSnapshot, removeCollector
- [ ] `NoOpDiagnosticSinkTest`: `record()` wirft keine Exception, Singleton-Semantik

### DoD 3: Tests — Integrationstests

- [ ] `DiagnosticControllerIntegrationTest`: POST config, GET snapshot, GET trace — alle Endpoints erreichbar und korrekt serialisiert
- [ ] `DiagnosticStreamIntegrationTest`: SSE-Stream liefert Events nach Config-Aenderung auf STANDARD
- [ ] Bestehende Tests bleiben GRUEN (keine Regression durch DiagnosticSink-Parameter in Konstruktoren)

### DoD 4: Frontend-Tests

- [ ] `DiagnosticsPanel.test.tsx`: Rendert korrekt, collapsed by default, expandable
- [ ] `VerbositySelector.test.tsx`: Alle 4 Optionen, onChange-Handler
- [ ] `FeatureToggleCheckboxes.test.tsx`: 6 Checkboxen, disabled bei OFF
- [ ] `DiagnosticTraceFeed.test.tsx`: Rendert Entries, FIFO-Verdraengung bei > 100

### DoD 5: ChatGPT-Review (drei Dimensionen)

- [ ] **Dimension 1 — Spec-Review:** Vergleich der Implementierung gegen Kap 0 (Architektur), Kap 6 (Rules Engine), Kap 9 (Frontend). Abweichungen identifizieren.
- [ ] **Dimension 2 — Code-Quality-Review:** Zero-Overhead-Garantie verifizieren, Thread-Safety des RingBuffers, korrekte Abhaengigkeitsrichtung (DiagnosticSink in odin-api), JavaDoc-Vollstaendigkeit.
- [ ] **Dimension 3 — Praxis-Review:** Sind die Default-Werte sinnvoll (buffer-capacity=500)? Ist die Ringbuffer-Semantik fuer einen Agenten brauchbar? Gibt es Szenarien (viele Pipelines, FULL-Modus, hohe Bar-Frequenz) bei denen der Speicherverbrauch problematisch wird?
- [ ] Findings bewertet und berechtigte Findings behoben

### DoD 6: Protokolldatei

- [ ] `protocol.md` in `T:/codebase/its_odin/temp/userstories/ODIN-057_agent-diagnostic-layer/` erstellt mit allen Pflichtabschnitten

### DoD 7: Abschluss

- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Temporaere Protokolldateien geloescht (Housekeeping-Regel aus MEMORY.md)

---

## Notizen fuer den Implementierer

### Kritischer Punkt: DiagnosticSink in odin-api vs. DiagnosticCollector in odin-core

Die zentrale architektonische Herausforderung ist die Abhaengigkeitsrichtung. `RulesEngine` (odin-brain) und `OrderManagementService` (odin-execution) duerfen NICHT von odin-core abhaengen. Die Loesung:

1. `DiagnosticSink` Interface in odin-api definieren (nur `record()`-Methode)
2. `DiagnosticCollector` in odin-core implementiert `DiagnosticSink`
3. `NoOpDiagnosticSink` in odin-api als Fallback fuer Tests
4. Fachmodule empfangen `DiagnosticSink` — sie wissen nicht, ob es ein Collector oder NoOp ist

Das ist dasselbe Pattern wie bei `MarketClock`, `EventLog`, `LlmAnalyst` — Port in odin-api, Implementierung in Fachmodul oder odin-core.

### Kritischer Punkt: Supplier-Pattern fuer Zero Overhead

Der `Supplier<Map<String, Object>>` ist KEIN Nice-to-Have, sondern architektonisch kritisch. Ohne Supplier wuerde jeder `record()`-Aufruf die Map erzeugen (String-Konkatenation, Boxing von Doubles, Map-Allokation) — auch im OFF-Modus. Mit dem Supplier-Pattern wird die Map NUR erzeugt, wenn der Entry tatsaechlich aufgezeichnet wird.

**Anti-Pattern (NICHT so implementieren):**
```java
// FALSCH: Map wird IMMER erzeugt, auch im OFF-Modus
collector.record(RULES, STANDARD, barIndex, "CHECK",
    Map.of("rsi", String.valueOf(rsi), "threshold", String.valueOf(threshold)));
```

**Korrekt:**
```java
// RICHTIG: Map wird NUR erzeugt, wenn record() entscheidet aufzuzeichnen
collector.record(RULES, STANDARD, barIndex, "CHECK",
    () -> Map.of("rsi", String.valueOf(rsi), "threshold", String.valueOf(threshold)));
```

### Kritischer Punkt: Bestehende Tests nicht brechen

Viele bestehende Unit-Tests instanziieren Pipeline-Komponenten (RulesEngine, DecisionArbiter, KpiEngine, OMS) direkt mit ihren Konstruktoren. Wenn ein neuer Parameter `DiagnosticSink` hinzugefuegt wird, brechen diese Tests.

**Loesung:** In jedem bestehenden Test `NoOpDiagnosticSink.INSTANCE` als Argument uebergeben. Das ist ein einmaliger, mechanischer Aufwand. Alternativ: einen Convenience-Konstruktor OHNE DiagnosticSink anbieten, der intern `NoOpDiagnosticSink.INSTANCE` verwendet — aber das verleitet dazu, die Instrumentierung zu vergessen. Entscheidung dem Implementierer ueberlassen, in `protocol.md` begruenden.

### DiagnosticRegistry Lifecycle

Die `DiagnosticRegistry` muss Pipeline-Deregistrierungen korrekt handhaben. Wenn eine Pipeline beendet wird (EOD, Kill-Switch), muss der zugehoerige Collector entfernt werden, um Memory-Leaks zu vermeiden. Der geeignete Hook ist `PipelineStateMachine.onStateChange(EOD)` oder die Pipeline-Teardown-Logik in `LifecycleManager`.

### SSE-Stream: Eigener Emitter oder bestehender SseEmitterManager?

Zwei Optionen:
1. **Eigener SseEmitterManager** fuer den Diagnostic-Stream (isoliert, kein Risiko fuer bestehende Streams)
2. **Bestehenden SseEmitterManager erweitern** um einen dritten Stream-Typ (`diagnostic`)

**Empfehlung:** Option 1 (eigener Manager). Begruendung: Der Diagnostic-Stream hat ein anderes Lastprofil (potentiell sehr viele Events bei FULL), und eine Stoerung darf die Monitoring-Streams nicht beeinflussen. Der Implementierer kann aber auch Option 2 waehlen, wenn die Implementierung sauberer ist — in `protocol.md` begruenden.

### Frontend: Collapsible Panel — kein eigener Router-Pfad

Das Diagnostics-Panel ist KEIN eigenstaendiger Screen, sondern ein collapsible Panel innerhalb bestehender Views (Backtest-Detail, Live-Dashboard). Es hat keinen eigenen Router-Pfad. Das Panel wird als React-Komponente in die bestehenden Views eingebettet — aehnlich wie die Chart-Annotation-Leiste oder das Alert-Feed.

### Config-Reset bei Backend-Restart

Die DiagnosticConfig wird In-Memory gehalten (AtomicReference). Bei Backend-Restart wird sie aus den Properties neu initialisiert (Default: OFF). Das ist beabsichtigt — ein Agent, der Diagnostics nutzt, muss sie nach einem Restart erneut aktivieren. Das ist konsistent mit dem Lean-Prinzip: kein persistierter Zustand fuer Diagnostics.
