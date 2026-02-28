# Story: ODIN-055 — LLM Sampling Policy & Real-LLM Backtest Mode

## Metadaten

| Feld | Wert |
|------|------|
| **Titel** | LLM Sampling Policy & Real-LLM Backtest Mode |
| **Modul** | odin-brain (primär: `LlmSamplingPolicy`, `LlmCallRecord`-Persistenz), odin-api (neu: `LlmCallRecord`-Record), odin-backtest (BacktestRunner: REAL_LLM-Modus) |
| **Phase** | 2 |
| **Abhängigkeiten** | Keine Vorläufer. ODIN-056 (Latency-Aware Replay) blockiert auf dieser Story. |
| **Geschätzter Umfang** | L |

---

## Kontext

### Ist-Zustand (Problem)

Backtests produzieren derzeit 0 Trades. Ursache: `CachedAnalyst` liefert bei Cache-Miss
`regime = UNCERTAIN` und `regimeConfidence = 0.0`. Da alle Backtest-Läufe mit neuen Daten
beginnen, für die kein Cache-Eintrag existiert, ist der LLM-Output dauerhaft
`regimeConfidence = 0.0`. Das `regimeConfidence < 0.5`-Gate in `RulesEngine` und `DecisionArbiter`
blockiert damit jeden Entry — der Backtest ist funktionslos.

Zusätzlich fehlt jede LLM-Sampling-Policy: `TradingPipeline.resolveLatestLlmAnalysis()` ruft
den LLM bei jedem Snapshot auf, sobald die Freshness-TTL abgelaufen ist — also potenziell 100x
pro Trading-Tag. Das ist weder skalierbar noch entspricht es der Architekturvorgabe aus Kap. 5
(volatilitätsabhängige Intervalle + event-driven Trigger).

### Soll-Zustand

**Real-LLM-Modus (erster Backtest-Lauf):**
- Der Backtest kann mit echter LLM-API betrieben werden (Konfiguration: `odin.brain.llm.backtest-provider=CLAUDE` oder `OPENAI`)
- LLM wird NICHT bei jedem Bar aufgerufen, sondern gemäß einer konfigurierbaren `LlmSamplingPolicy`:
  - Zeitbasiert: alle N abgeschlossenen Bars (Standardwert: alle 5 Bars = ~5 Minuten bei 1m-Bars)
  - Eventgetrieben: bei signifikanter Preisbewegung > X% innerhalb einer Bar (Standardwert: 1.5%)
  - Composite (OR): Mindestfrequenz + Event-Trigger (Standardkonfiguration)
- Jede LLM-Antwort wird als `LlmCallRecord` in der DB persistiert (Tabelle `llm_call_record`)
- `LlmCallRecord` enthält u.a. `requestMarketTime`, `latencyMs`, `analysis` (JSON)

**Replay-Modus (Folgeläufe, ODIN-056):**
- `CachedAnalyst` liest `LlmCallRecord` aus der DB und liefert gecachte Antworten zeitverzögert
- Wird in ODIN-056 implementiert — diese Story liefert das Datenbankfundament

### Warum diese Reihenfolge

`LlmCallRecord` (Datenbankschema + Entity + Repository) ist das Fundament, auf dem der
latenzfähige Replay-Mechanismus in ODIN-056 aufbaut. ODIN-055 implementiert das Schema und
befüllt die Tabelle im REAL_LLM-Modus. ODIN-056 liest daraus.

---

## Scope

### In Scope

**odin-api:**
- Neues Record `LlmCallRecord` als API-DTO/Domänenobjekt (in `de.its.odin.api.dto`)

**odin-brain:**
- `LlmSamplingPolicy` — Interface + Implementierungen:
  - `TimeBasedSamplingPolicy` — feuert alle N abgeschlossenen Bars
  - `EventDrivenSamplingPolicy` — feuert bei Preisbewegung > X% in einer Bar
  - `CompositeSamplingPolicy` — OR-Verknüpfung: feuert wenn eine der beiden Policies triggered
- `LlmCallRecordEntity` — JPA-Entity (in `de.its.odin.brain.persistence`)
- `LlmCallRecordRepository` — JPA-Repository
- Integration in `TradingPipeline.resolveLatestLlmAnalysis()`: Sampling-Policy-Check vor LLM-Call; nach erfolgreichem Call `LlmCallRecord` persistieren

**odin-persistence:**
- Flyway-Migration V0XX: neue Tabelle `llm_call_record`

**odin-backtest:**
- `BacktestRunner`: REAL_LLM-Modus ist bereits technisch vorhanden (via `backtestProvider = CLAUDE/OPENAI`), aber bisher ruft jede Pipeline den LLM bei jedem Snapshot auf
- Übergabe der `LlmSamplingPolicy` an `PipelineFactory` für Backtest-Runs
- Konfiguration: `odin.backtest.llm.sampling-mode=REAL_LLM` (Alias für `backtestProvider != CACHED`) — kein neuer Config-Key nötig, realer LLM-Provider ist bereits konfigurierbar

**Konfiguration:**
- `odin.brain.llm.sampling.time-interval-bars=5`
- `odin.brain.llm.sampling.event-threshold-pct=1.5`

### Out of Scope

- Latency-Aware Replay (`CachedAnalyst` liest aus `llm_call_record`) — das ist ODIN-056
- Frontend-Änderungen
- Beschleunigung der SimClock über echte Marktzeit hinaus
- Änderungen am LLM-Schema (Output-Format bleibt unverändert)

---

## Akzeptanzkriterien

- [ ] **AC-1:** `LlmSamplingPolicy`-Interface existiert in `odin-brain`. `TimeBasedSamplingPolicy`, `EventDrivenSamplingPolicy` und `CompositeSamplingPolicy` implementieren es. Unit-Tests decken alle drei Policies ab.
- [ ] **AC-2:** `CompositeSamplingPolicy` ist die Default-Policy in der Backtest-Konfiguration (beide Teilpolicies konfigurierbar).
- [ ] **AC-3:** `TradingPipeline.resolveLatestLlmAnalysis()` konsultiert `LlmSamplingPolicy` vor jedem LLM-Call. Bei `shouldSample() == false` wird der letzte bekannte LLM-Output zurückgegeben — kein neuer API-Call.
- [ ] **AC-4:** Der letzte bekannte LLM-Output wird zwischen Sampling-Triggern wiederverwendet, auch wenn er älter als `entryFreshnessMaxS` ist. **Kein null-Return und kein UNCERTAIN-Default** zwischen Triggern.
- [ ] **AC-5:** Nach einem erfolgreichen LLM-Call wird ein `LlmCallRecord` in der Tabelle `llm_call_record` persistiert. Der Record enthält mindestens: `instrumentId`, `runId`, `requestMarketTime`, `responseMarketTime`, `latencyMs`, `analysis` (JSON).
- [ ] **AC-6:** Die Tabelle `llm_call_record` existiert nach Flyway-Migration. Schema entspricht der Spezifikation in den Technischen Details.
- [ ] **AC-7:** Im REAL_LLM-Backtest-Modus (z.B. `backtestProvider=CLAUDE`) wird der LLM NICHT bei jedem Bar aufgerufen, sondern nur gemäß `LlmSamplingPolicy`. Verifikation durch Log-Ausgabe oder Unit-Test.
- [ ] **AC-8:** `DegradationManager` eskaliert im Backtest NICHT in `QUANT_ONLY` allein deswegen, weil am Anfang kein `LlmCallRecord` vorhanden ist. Das Fehlen eines initialen Records ist erwartetes Verhalten bis zum ersten Sampling-Trigger.
- [ ] **AC-9:** `TimeBasedSamplingPolicy` ist korrekt konfigurierbar über `odin.brain.llm.sampling.time-interval-bars`. `EventDrivenSamplingPolicy` über `odin.brain.llm.sampling.event-threshold-pct`. Properties existieren mit sinnvollen Defaults.

---

## Technische Details

### Neues Record: `LlmCallRecord` (odin-api)

Paket: `de.its.odin.api.dto`

```java
/**
 * Immutable data transfer object for a persisted LLM call.
 * Used to build the replay cache in simulation mode (Latency-Aware CachedAnalyst, ODIN-056).
 *
 * @param id                  surrogate primary key (UUID)
 * @param instrumentId        pipeline instrument identifier
 * @param runId               trading run join key
 * @param backtestId          backtest batch identifier (null for live runs)
 * @param requestMarketTime   SimClock/MarketClock time when the LLM request was initiated
 * @param responseMarketTime  SimClock time when the response became available
 *                            (requestMarketTime + latencyMs in market time)
 * @param latencyMs           actual wall-clock latency of the provider API call in milliseconds
 * @param analysis            full JSON-serialized LlmAnalysis (or LlmTacticalOutput) payload
 * @param promptVersion       prompt schema version (e.g. "1.1.0")
 * @param modelId             LLM model identifier (e.g. "claude-sonnet-4-5-20250929")
 */
public record LlmCallRecord(
        UUID id,
        String instrumentId,
        String runId,
        String backtestId,
        Instant requestMarketTime,
        Instant responseMarketTime,
        long latencyMs,
        String analysis,
        String promptVersion,
        String modelId
) {}
```

### Interface: `LlmSamplingPolicy` (odin-brain)

Paket: `de.its.odin.brain.llm.sampling`

```java
/**
 * Determines whether the system should call the LLM at the current decision bar.
 * Applied before every potential LLM call in {@code TradingPipeline.resolveLatestLlmAnalysis()}.
 *
 * <p>Implementations must be stateful: they track the last trigger time/bar to compute
 * elapsed intervals correctly.
 *
 * @see TimeBasedSamplingPolicy
 * @see EventDrivenSamplingPolicy
 * @see CompositeSamplingPolicy
 */
public interface LlmSamplingPolicy {

    /**
     * Evaluates whether an LLM call should be made for the current snapshot.
     *
     * @param snapshot         current market snapshot (contains currentPrice, previousClose, marketTime)
     * @param barIndex         monotonically increasing bar counter since pipeline start (0-based)
     * @param lastSampleBarIndex bar index of the last LLM call, or -1 if none
     * @return true if the LLM should be called now
     */
    boolean shouldSample(MarketSnapshot snapshot, long barIndex, long lastSampleBarIndex);

    /**
     * Notifies the policy that a sample was taken at the given bar index.
     * Must be called after a successful LLM call.
     *
     * @param barIndex bar index when the sample was taken
     */
    void onSampleTaken(long barIndex);
}
```

### Implementierungen (odin-brain, Paket: `de.its.odin.brain.llm.sampling`)

**`TimeBasedSamplingPolicy`:**
- Konstruktor: `(int intervalBars)` — aus Properties `odin.brain.llm.sampling.time-interval-bars`
- `shouldSample()`: `true` wenn `barIndex - lastSampleBarIndex >= intervalBars` ODER `lastSampleBarIndex == -1` (erster Call)
- Thread-Sicherheit: Pro-Pipeline-POJO, kein Threading erforderlich

**`EventDrivenSamplingPolicy`:**
- Konstruktor: `(double eventThresholdPct)` — aus Properties `odin.brain.llm.sampling.event-threshold-pct`
- `shouldSample()`: `true` wenn `|snapshot.currentPrice() - snapshot.previousBarClose()| / snapshot.previousBarClose() * 100 >= eventThresholdPct`
- Fallback bei null/NaN: `false` (kein Trigger wenn keine valide Preisbewegung berechenbar)
- `onSampleTaken()`: No-op (stateless bezüglich Bar-Index)

**`CompositeSamplingPolicy`:**
- Konstruktor: `(LlmSamplingPolicy timeBased, LlmSamplingPolicy eventDriven)`
- `shouldSample()`: `timeBased.shouldSample(...) || eventDriven.shouldSample(...)`
- `onSampleTaken()`: delegiert an beide Teil-Policies

### JPA-Entity: `LlmCallRecordEntity` (odin-brain)

Paket: `de.its.odin.brain.persistence`

```java
@Entity
@Table(name = "llm_call_record")
public class LlmCallRecordEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(name = "id", nullable = false, updatable = false)
    private UUID id;

    @Column(name = "instrument_id", nullable = false, length = 20)
    private String instrumentId;

    @Column(name = "run_id", nullable = false, length = 36)
    private String runId;

    @Column(name = "backtest_id", length = 36)
    private String backtestId;                  // null für Live-Runs

    @Column(name = "request_market_time", nullable = false)
    private Instant requestMarketTime;

    @Column(name = "response_market_time", nullable = false)
    private Instant responseMarketTime;

    @Column(name = "latency_ms", nullable = false)
    private long latencyMs;

    @Column(name = "analysis", nullable = false, columnDefinition = "text")
    private String analysis;                    // JSON, vollständige LLM-Antwort

    @Column(name = "prompt_version", nullable = false, length = 20)
    private String promptVersion;

    @Column(name = "model_id", nullable = false, length = 100)
    private String modelId;

    // JPA-Anforderung: No-Arg-Konstruktor (protected genügt)
    protected LlmCallRecordEntity() {}

    // All-args public Konstruktor für programmatische Erzeugung
    public LlmCallRecordEntity(String instrumentId, String runId, String backtestId,
                                Instant requestMarketTime, Instant responseMarketTime,
                                long latencyMs, String analysis,
                                String promptVersion, String modelId) { ... }

    // Getter für alle Felder (keine Setter — Entity ist nach Persistierung immutabel)
}
```

### Repository: `LlmCallRecordRepository` (odin-brain)

Paket: `de.its.odin.brain.persistence`

```java
public interface LlmCallRecordRepository extends JpaRepository<LlmCallRecordEntity, UUID> {

    /**
     * Finds all LLM call records for a specific instrument and run, ordered by
     * requestMarketTime ascending. Used by the latency-aware replay in ODIN-056.
     */
    List<LlmCallRecordEntity> findByInstrumentIdAndRunIdOrderByRequestMarketTimeAsc(
            String instrumentId, String runId);

    /**
     * Finds all LLM call records for a backtest batch (multiple runs of the same backtest),
     * ordered by requestMarketTime ascending. Used by ODIN-056 for cross-run replay.
     */
    List<LlmCallRecordEntity> findByBacktestIdAndInstrumentIdOrderByRequestMarketTimeAsc(
            String backtestId, String instrumentId);
}
```

### Datenbankschema: Flyway-Migration (odin-persistence)

Migration-Datei: `V0XX__create_llm_call_record.sql`

```sql
CREATE TABLE llm_call_record (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id        VARCHAR(20)  NOT NULL,
    run_id               VARCHAR(36)  NOT NULL,
    backtest_id          VARCHAR(36),                             -- NULL für Live-Runs
    request_market_time  TIMESTAMPTZ  NOT NULL,
    response_market_time TIMESTAMPTZ  NOT NULL,
    latency_ms           BIGINT       NOT NULL,
    analysis             TEXT         NOT NULL,                   -- vollständiger JSON-Payload
    prompt_version       VARCHAR(20)  NOT NULL,
    model_id             VARCHAR(100) NOT NULL,
    created_at           TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- Index für Replay-Abfragen: Suche nach backtestId + instrumentId + requestMarketTime
CREATE INDEX idx_llm_call_record_backtest_instrument_time
    ON llm_call_record (backtest_id, instrument_id, request_market_time);

-- Index für Run-spezifische Abfragen
CREATE INDEX idx_llm_call_record_run_instrument
    ON llm_call_record (run_id, instrument_id);
```

### Integration in `TradingPipeline.resolveLatestLlmAnalysis()`

Die Methode erhält eine neue Parametrierung. Da `TradingPipeline` kein Spring-Bean ist,
wird `LlmSamplingPolicy` via `PipelineFactory`-Konstruktor hereingereicht.

**Neue Logik (Pseudocode):**

```
resolveLatestLlmAnalysis(MarketSnapshot snapshot):

    1. QUANT_ONLY-Check wie bisher → early return null

    2. Freshness-Check entfällt als LLM-Trigger. Stattdessen:
       if samplingPolicy.shouldSample(snapshot, currentBarIndex, lastSampleBarIndex):
           → tatsächlichen LLM-Call starten
       else:
           → lastLlmAnalysis zurückgeben (kann älter als entryFreshnessMaxS sein!)
              Wenn lastLlmAnalysis == null (allererster Bar, noch kein Call):
              → null zurückgeben (DegradationManager NICHT eskalieren — erwartet!)

    3. Bei LLM-Call:
       - wallClockStart = System.currentTimeMillis()
       - freshAnalysis = llmAnalyst.analyze(snapshot, runContext)
       - wallClockEnd = System.currentTimeMillis()
       - latencyMs = wallClockEnd - wallClockStart
       - lastLlmAnalysis = freshAnalysis
       - samplingPolicy.onSampleTaken(currentBarIndex)
       - lastSampleBarIndex = currentBarIndex

    4. LlmCallRecord persistieren (async, non-blocking für Live-Mode):
       - requestMarketTime = snapshot.marketTime()
       - responseMarketTime = marketClock.now()
       - analysis = Json.serialize(freshAnalysis)
       → llmCallRecordRepository.save(entity)    [im Backtest synchron, im Live async]

    5. degradationManager.onLlmSuccess()
    6. return freshAnalysis
```

**Wichtige Designentscheidungen:**

- **Freshness-Gate entfällt als Trigger:** Das Freshness-Gate in `DecisionArbiter`
  (Kap. 5 §4) bleibt bestehen — es entscheidet ob die gecachte Analyse für Entry-Entscheidungen
  akzeptiert wird. Aber die `resolveLatestLlmAnalysis()`-Methode ruft den LLM nicht mehr bei
  TTL-Überschreitung, sondern ausschliesslich gemäss `LlmSamplingPolicy`.
- **Null-Rückgabe nur am Anfang:** Bevor der erste Sampling-Trigger feuert, gibt die Methode
  `null` zurück. Das ist akzeptables Verhalten — die RulesEngine und der Arbiter behandeln null
  bereits korrekt. `DegradationManager.onLlmFailure()` wird in diesem Fall NICHT aufgerufen.
- **Persistenz-Scope:** Im Backtest ist `LlmCallRecordRepository` ein Spring-Bean (via odin-app).
  `PipelineFactory` übergibt das Repository als `Optional<LlmCallRecordRepository>` — wenn null
  (Tests, CLI-Modus), wird kein Record persistiert.

### `PipelineFactory`-Erweiterung

`PipelineFactory` bekommt einen neuen optionalen Parameter:
```java
// Neuer Parameter (kann null sein wenn kein Repository verfügbar)
@Nullable LlmCallRecordRepository llmCallRecordRepository
```

Intern erzeugt `PipelineFactory` die `LlmSamplingPolicy` aus `BrainProperties.LlmSamplingProperties`.

### Neue Konfigurationsstruktur (odin-brain.properties)

```properties
# LLM Sampling Policy
odin.brain.llm.sampling.time-interval-bars=5
odin.brain.llm.sampling.event-threshold-pct=1.5
```

Die Properties werden als neues nested Record `LlmSamplingProperties` in `BrainProperties.LlmProperties` ergänzt:

```java
/**
 * LLM sampling policy configuration for backtest and live modes.
 *
 * @param timeIntervalBars   number of bars between time-based LLM samples (default 5)
 * @param eventThresholdPct  intra-bar price movement percentage threshold for event-driven trigger (default 1.5)
 */
public record LlmSamplingProperties(
        @Min(1) int timeIntervalBars,
        @DecimalMin("0.0") double eventThresholdPct
) {}
```

Das neue Record wird in `LlmProperties` eingebettet:
```java
public record LlmProperties(
        // ... alle bestehenden Felder ...
        @NotNull @Valid LlmSamplingProperties sampling
) {}
```

---

## Konzept-Referenzen

| Dokument | Abschnitt |
|----------|-----------|
| `docs/backend/architecture/05-llm-integration.md` | Kap. 7: Aufruf-Strategie, Trigger-Typen, periodisches Intervall |
| `docs/backend/architecture/05-llm-integration.md` | Kap. 10: Simulation-Semantik, CachedAnalyst, LLM-Timing-Modell |
| `docs/backend/architecture/06-rules-engine.md` | Kap. 4: Freshness-Check, Default-Werte bei fehlendem LLM-Output |
| `docs/concept/04-llm-integration.md` | Sampling/Cadence-Mechanismus |
| `docs/backend/architecture/08-data-model.md` | Entity-Konventionen, Flyway-Migration |

---

## Guardrail-Referenzen

| Guardrail | Pfad |
|-----------|------|
| Entwicklungsprozess | `docs/backend/guardrails/development-process.md` |
| Modulstruktur | `docs/backend/guardrails/module-structure.md` |
| CLAUDE.md | `T:/codebase/its_odin/CLAUDE.md` |

---

## Definition of Done

### 2.1 Code-Qualität
- [ ] Implementierung vollständig gemäß Akzeptanzkriterien
- [ ] `mvn compile -pl odin-api,odin-brain,odin-backtest,odin-app --also-make` fehlerfrei
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records für DTOs (`LlmCallRecord`)
- [ ] JavaDoc auf allen neuen/geänderten public Klassen und Methoden
- [ ] Keine TODO/FIXME-Kommentare im abgelieferten Code
- [ ] `LlmSamplingPolicy`-Implementierungen sind Pro-Pipeline-POJOs ohne Spring-Annotations
- [ ] `LlmCallRecordEntity` trägt Naming-Konvention `*Entity`
- [ ] `MarketClock.now()` im Trading-Codepfad, kein `Instant.now()` oder `System.currentTimeMillis()`
  (Ausnahme: Wall-Clock-Latenz-Messung darf `System.currentTimeMillis()` nutzen — das ist keine
  Trading-Zeitlogik, sondern reine Telemetrie)

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] `TimeBasedSamplingPolicyTest`: `shouldSample()` bei erstem Call, nach Intervall, vor Intervall
- [ ] `EventDrivenSamplingPolicyTest`: Trigger bei Bewegung >= Schwelle, kein Trigger darunter, Null-Safety
- [ ] `CompositeSamplingPolicyTest`: OR-Logik, beide Delegates werden korrekt delegiert
- [ ] `LlmSamplingPolicyTest` (Integration): vollständiger Ablauf mit Backtest-ähnlichem Bar-Stream
  → Verifikation dass LLM-Calls nur bei Policy-Triggern erfolgen

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] `LlmCallRecordRepositoryIntegrationTest` (Zonky Embedded PostgreSQL oder H2):
  - Record speichern → wieder laden
  - `findByInstrumentIdAndRunIdOrderByRequestMarketTimeAsc` liefert korrekte Sortierung
  - `findByBacktestIdAndInstrumentIdOrderByRequestMarketTimeAsc` filtert korrekt
- [ ] `BrainPropertiesIntegrationTest` (bestehend erweitern): neue `sampling`-Properties werden korrekt gebunden

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT nach Edge-Cases gefragt:
  - Was passiert wenn `previousBarClose = 0` in `EventDrivenSamplingPolicy`?
  - Ist die `onSampleTaken()`-Semantik korrekt wenn beide Policies in `CompositeSamplingPolicy` triggern?
  - Ist null-Return vor erstem Sampling-Trigger semantisch korrekt (DegradationManager-Auswirkung)?
  - Race Condition: Was wenn SimClock zwischen `requestMarketTime` und `responseMarketTime` sprunghaft voranschreitet?
- [ ] Findings bewertet, berechtigte Findings eingearbeitet
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch ChatGPT (drei Dimensionen)

Da Gemini-Pool dauerhaft ausgefallen ist, deckt ChatGPT alle drei Review-Dimensionen ab.

- [ ] **Dimension 1 — Spec-Review:** Vergleich der Implementierung gegen Kap. 5 (LLM-Integration),
  Kap. 7 (Aufruf-Strategie, Trigger-Typen), Kap. 8 (Datenmodell). Abweichungen identifizieren.
- [ ] **Dimension 2 — Code-Quality-Review:** Null-Safety in allen drei Policies, Thread-Safety
  der pro-Pipeline-POJOs, korrekte Entity-Konventionen, JavaDoc-Vollständigkeit.
- [ ] **Dimension 3 — Praxis-Review:** Sind die Default-Werte sinnvoll (5 Bars, 1.5%)? Gibt es
  Szenarien (z.B. sehr schnelle Märkte, sehr hohe Volatilität) bei denen die Policy kontraproduktiv ist?
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` in `T:/codebase/its_odin/temp/userstories/ODIN-055_llm-sampling-policy/` erstellt
  mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekräftiger Message
- [ ] Push auf Remote
- [ ] Temporäre Protokolldateien gelöscht (Housekeeping-Regel aus MEMORY.md)

---

## Notizen für den Implementierer

### Kritischer Punkt: `DegradationManager` und null-Return

Die aktuelle `resolveLatestLlmAnalysis()`-Methode ruft `degradationManager.onLlmFailure()`
wenn der LLM-Call scheitert. Sie ruft es aber NICHT wenn gar kein Call stattfindet.
Mit der neuen Sampling-Policy ist das explizit gewünscht: Vor dem ersten Sampling-Trigger
gibt die Methode `null` zurück **ohne** `onLlmFailure()` zu rufen. Der `DegradationManager`
darf dadurch NICHT in `QUANT_ONLY` wechseln.

Formulierung: "Kein LLM-Call = kein Failure. Failure = Call wurde gemacht und ist gescheitert."

### Kritischer Punkt: Freshness-Gate bleibt gültig

Das Freshness-Gate in `DecisionArbiter` und `RulesEngine` (Kap. 5 §4) bleibt unverändert
aktiv. Die gecachte `LlmAnalysis` kann für Entry-Entscheidungen als "stale" gelten,
wenn sie älter als `entryFreshnessMaxS` ist. Das ist korrekt — das System fällt dann
auf `regime = UNCERTAIN, regimeConfidence = 0.0` zurück und blockiert Entries. Das ist
kein Bug, sondern Feature: Die Sampling-Policy verhindert unnötige API-Calls; das
Freshness-Gate verhindert Entries auf Basis veralteter LLM-Daten.

**Konsequenz für Konfiguration:** `time-interval-bars=5` muss kleiner als
`entryFreshnessMaxS / 60` (Bars pro Minute = 1 für 1m-Bars) sein. Bei
`entryFreshnessMaxS=120` bedeutet das max. 2 Bars. Bei `time-interval-bars=5`
können also Entry-Opportunities durch das Freshness-Gate blockiert werden.
Der Implementierer soll das in `protocol.md` dokumentieren und einen sinnvollen
Default vorschlagen — kein eigenmächtiges Ändern der Architektur.

### Modul-Abhängigkeiten beachten

`LlmCallRecordEntity` und `LlmCallRecordRepository` leben in `odin-brain`.
Das ist korrekt (DDD: Entity gehört zur Brain-Domäne). `odin-persistence`
enthält keine fachlichen Entities — nur DataSource, JPA-Config, Flyway.
Bei `@EntityScan` und `@EnableJpaRepositories` in `odin-app` muss das
neue Package `de.its.odin.brain.persistence` enthalten sein.
**Vor der Implementierung prüfen, ob das Package bereits gescannt wird.**

### Serialisierung des `analysis`-Felds

Das `analysis`-Feld in `LlmCallRecordEntity` speichert die vollständige
LLM-Antwort als JSON-String. Die Serialisierung muss deterministisch sein
(Reihenfolge der Felder stabil), damit ODIN-056 den Cache-Key korrekt
berechnen kann. Jackson mit `ObjectMapper` und
`MapperFeature.SORT_PROPERTIES_ALPHABETICALLY` verwenden oder explizite
Feldordnung sicherstellen. Details in ODIN-056 spezifiziert.

### `backtestId` befüllen

Im `BacktestRunner` ist `batchId` der übergreifende Identifier für einen
Multi-Day-Backtest. Dieser `batchId` soll als `backtestId` im `LlmCallRecord`
gespeichert werden, damit ODIN-056 alle Records eines Backtests laden kann.
Der `batchId` muss also von `BacktestRunner.runSingleDay()` bis in
`TradingPipeline` gelangen — entweder über `RunContext` oder über
`LlmCallRecordRepository` + direkte Übergabe. Empfohlen: `RunContext` um
ein Feld `batchId` erweitern (Koordination mit Architekt prüfen, da das
eine odin-api-Änderung ist; `RunContext` liegt in odin-api).
**Alternativ:** `backtestId` im `TradingPipeline`-Konstruktor als separaten
String-Parameter übergeben. Entscheidung dem Implementierer überlassen,
aber in `protocol.md` begründen.

### Wall-Clock vs. MarketClock für Latenz

`latencyMs` misst die tatsächliche API-Latenz in Wall-Clock-Zeit
(für Backtest-Simulationszwecke: wie lange hat der echte API-Call gedauert).
Das ist KEINE Trading-Zeitlogik, sondern Telemetrie. `System.currentTimeMillis()`
ist hier korrekt und erlaubt. `MarketClock.now()` für `requestMarketTime`
und `responseMarketTime` bleibt Pflicht — das sind Simulations-Zeitpunkte.
