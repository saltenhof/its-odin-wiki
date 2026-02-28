# Story: ODIN-056 — Latency-Aware CachedAnalyst & Fast Replay Mode

## Metadaten

| Feld | Wert |
|------|------|
| **Titel** | Latency-Aware CachedAnalyst & Fast Replay Mode |
| **Modul** | odin-brain (primär: `CachedAnalyst` überarbeiten), odin-backtest (`BacktestRunner`: REPLAY-Modus), odin-core (`SimClock`-Integration) |
| **Phase** | 2 |
| **Abhängigkeiten** | **ODIN-055 muss vollständig abgeschlossen sein** (Tabelle `llm_call_record` muss existieren, `LlmCallRecord`, `LlmCallRecordEntity`, `LlmCallRecordRepository` müssen vorhanden und compiliert sein) |
| **Geschätzter Umfang** | M |

---

## Kontext

### Ist-Zustand (Problem)

Nach Abschluss von ODIN-055 kann ein erster Backtest im REAL_LLM-Modus durchgeführt werden.
Dabei werden echte API-Calls gemacht und die Ergebnisse als `LlmCallRecord` in der Datenbank
persistiert. Folgende Probleme bleiben aber noch offen:

1. **Folgeläufe kosten Geld:** Jeder weitere Backtest mit identischen Marktdaten
   (z.B. zum Tunen von Quant-Parametern) würde erneut echte API-Calls machen.
   Das ist weder kosteneffizient noch skalierbar für iterative Optimierungsläufe.

2. **`CachedAnalyst` ignoriert Latenz:** Der aktuelle `CachedAnalyst` liefert bei Cache-Hit
   die Analyse sofort zurück (`latencyMs = 0`). Das ist für echte Replay-Semantik unzureichend:
   Im echten Handelsbetrieb hat der LLM typischerweise 1–5 Sekunden Latenz. Wenn diese Latenz
   nicht simuliert wird, kann der Backtest Entry-Entscheidungen auf Basis von LLM-Output treffen,
   der in der Realität noch nicht verfügbar gewesen wäre — das ist Look-Ahead-Bias.

3. **`CachedAnalyst` gibt bei Cache-Miss `confidence = 0.0` zurück:** Das blockiert alle
   Entries. Im REPLAY-Modus (nach ODIN-055) sollen `LlmCallRecord`-Einträge aus der DB als
   Quelle dienen — wenn kein Record verfügbar ist (z.B. am Anfang, vor dem ersten LLM-Trigger),
   soll `null` zurückgegeben werden, nicht `confidence = 0.0`.

### Soll-Zustand

**REPLAY-Modus (Folgeläufe):**
- `BacktestRunner` erkennt, dass für eine bestimmte `backtestId` bereits `LlmCallRecord`-Einträge
  in der DB vorhanden sind
- `CachedAnalyst` wird mit den geladenen Records initialisiert (einmalig beim Start des Tages)
- Bei jedem LLM-Analyse-Aufruf: suche den neuesten `LlmCallRecord` für dieses Instrument
  mit `responseMarketTime <= MarketClock.now()`
- Latenz ist korrekt simuliert: Wenn der SimClock-Zeitpunkt vor `responseMarketTime` liegt,
  ist die Analyse noch nicht verfügbar — `analyze()` gibt `null` zurück
- Wenn kein Record verfügbar (echte Vorab-Lücke): `null` zurückgeben (korrekt — NICHT `confidence = 0.0`)
- Backtests laufen mit voller SimClock-Geschwindigkeit (kein API-Call-Overhead)

**Kernprinzip:**
> Ein `LlmCallRecord` ist erst verfügbar wenn `SimClock.now() >= record.responseMarketTime()`.
> Das simuliert die echte Latenz des API-Calls korrekt.

### Nutzen

- Quant-Parameter können iterativ getunt werden (auch mit ML-Optimierungen) ohne LLM-Kosten
  zu wiederholen. LLM-Output bleibt konstant, Quant-Parameter variieren.
- Reproduzierbare Backtests mit echter Latenz-Semantik (kein Look-Ahead-Bias)
- `DegradationManager`-Verhalten in Replay-Modus korrekt konfiguriert: kein `QUANT_ONLY`
  am Anfang des Tages wenn kein erster Record verfügbar ist

---

## Scope

### In Scope

**odin-brain — `CachedAnalyst` überarbeiten:**
- `CachedAnalyst` bekommt `MarketClock`-Zugriff (Konstruktor-Injection)
- Neuer Konstruktor-Parameter: `MarketClock marketClock`
- Neuer Initialisierungs-Pfad: `preloadFromRecords(List<LlmCallRecord> records)` — lädt
  Records aus DB, sortiert nach `requestMarketTime` ascending
- `analyze(snapshot, context)` überarbeiten:
  - Suche neuesten Record mit `responseMarketTime <= marketClock.now()`
  - Gefunden: deserialisiere `analysis`-JSON zu `LlmAnalysis`, gib zurück (mit `cacheHit = true`)
  - Nicht gefunden: gib `null` zurück — NICHT mehr das UNCERTAIN-Default-Objekt

**odin-brain — `DegradationManager`-Integration im Replay-Modus:**
- Im REPLAY-Modus: `TradingPipeline.resolveLatestLlmAnalysis()` darf bei `null`-Rückgabe
  von `CachedAnalyst` NICHT `degradationManager.onLlmFailure()` aufrufen
- Die Pipeline muss zwischen "LLM-Call gescheitert" und "noch kein Record verfügbar" unterscheiden
- Umsetzung: neues Flag `replayMode` in `TradingPipeline`, oder der `CachedAnalyst` selbst
  signalisiert "not yet available" vs. "failure" (z.B. via checked Exception oder distinguishable null)
- Entscheidung: Empfohlen ist ein `Optional<LlmAnalysis>`-Rückgabetyp für eine neue interne
  Hilfsmethode — der Port `LlmAnalyst` bleibt unverändert (`LlmAnalysis analyze(...)`)

**odin-backtest — `BacktestRunner` REPLAY-Modus:**
- Beim Start eines Backtest-Tags: prüfe ob bereits `LlmCallRecord`-Einträge für `backtestId` +
  `instrumentId` + `tradingDate` vorhanden sind
- Wenn ja (REPLAY-Modus): lade Records, übergebe an `CachedAnalyst.preloadFromRecords()`
- Wenn nein (REAL_LLM-Modus): weiter wie bisher (echter API-Call gemäß ODIN-055-Logik)
- Auto-Detection: Das System erkennt automatisch ob REPLAY oder REAL_LLM — kein manueller
  Konfigurationsschalter nötig. Erste Läufe sind immer REAL_LLM, Folgeläufe immer REPLAY.

**SimClock-Integration:**
- `SimClock.now()` läuft bereits deterministisch mit Bar-Tempo (kein Änderungsbedarf)
- Keine Änderung an `SimClock` erforderlich — die Latenz-Semantik ergibt sich aus dem
  Vergleich `responseMarketTime <= simClock.now()` beim Abrufen des Records

**odin-brain — `LlmCallRecord`-Deserialisierung:**
- Neue Klasse `LlmCallRecordDeserializer` (odin-brain, `de.its.odin.brain.llm`)
- Verantwortlich für: JSON-String aus `LlmCallRecord.analysis()` → `LlmAnalysis`-Record
- Muss stabil sein gegenüber Schema-Unterschieden (defensives Parsing)

### Out of Scope

- Echte SimClock-Beschleunigung (Fast-Forward über Marktzeit) — SimClock läuft bereits
  mit Bar-Tempo, das ist ausreichend für den Replay-Modus
- Frontend-Änderungen
- Änderungen am LLM-Schema oder Prompt-Format
- Multi-Backtest-Vergleich auf Basis verschiedener LLM-Record-Sets
- Automatische Invalidierung des Record-Cache bei Prompt-Version-Änderungen
  (das ist in ODIN-055 bereits durch `promptVersion`-Feld im Record vorbereitet, wird aber
  hier noch nicht implementiert — Verhalten bei Version-Mismatch: Warnung loggen, Records nutzen)

---

## Akzeptanzkriterien

- [ ] **AC-1:** `CachedAnalyst` hat einen Konstruktor der `MarketClock` als Parameter akzeptiert.
  Die bestehende `CachedAnalyst()`-Konstruktur (no-arg) wird durch den neuen ersetzt oder
  deprecated — kein Mischbetrieb.
- [ ] **AC-2:** `CachedAnalyst.analyze()` gibt `null` zurück wenn kein `LlmCallRecord` mit
  `responseMarketTime <= marketClock.now()` gefunden wird. Es wird KEIN
  `Regime.UNCERTAIN, confidence=0.0`-Default-Objekt mehr zurückgegeben.
- [ ] **AC-3:** Bei Cache-Hit (Record gefunden, `responseMarketTime` erreicht) gibt `CachedAnalyst`
  die deserialisierte `LlmAnalysis` korrekt zurück. `cacheHit = true`, `latencyMs = record.latencyMs()`.
- [ ] **AC-4:** `BacktestRunner` erkennt automatisch ob ein REPLAY-Modus möglich ist:
  Wenn für `backtestId + instrumentId + tradingDate` mindestens ein `LlmCallRecord` in der DB
  existiert, wird der REPLAY-Modus aktiviert. Andernfalls: REAL_LLM-Modus.
- [ ] **AC-5:** Im REPLAY-Modus werden Records einmalig beim Start des Trading-Tags geladen
  (kein wiederholtes DB-Query pro Bar).
- [ ] **AC-6:** `TradingPipeline.resolveLatestLlmAnalysis()` ruft `degradationManager.onLlmFailure()`
  im REPLAY-Modus NICHT auf, wenn `CachedAnalyst.analyze()` `null` zurückgibt. Unterscheidung
  zwischen "noch kein Record verfügbar" und "echter LLM-Fehler" ist korrekt implementiert.
- [ ] **AC-7:** Ein Backtest-Replay-Lauf produziert dieselben Entry/Exit-Entscheidungen wie der
  ursprüngliche REAL_LLM-Lauf (wenn alle anderen Parameter identisch sind). Verifikation durch
  Integrationstest oder manuelle Log-Inspektion.
- [ ] **AC-8:** `LlmCallRecordDeserializer` ist defensiv implementiert: Bei unbekannten JSON-Feldern
  wird geloggt (WARN), die Deserialisierung aber nicht abgebrochen. Bei fehlenden Pflichtfeldern
  (z.B. `regime`) wird `null` zurückgegeben und geloggt.
- [ ] **AC-9:** Ein vollständiger Replay-Backtest-Lauf (2+ Tage) produziert mindestens so viele
  Trades wie der REAL_LLM-Originallauf. Der Wert darf nicht kleiner sein (keine zusätzlichen
  Blockierungen durch fehlerhafte Latenz-Semantik).

---

## Technische Details

### `CachedAnalyst` — überarbeitete Implementierung

**Neue Felder:**

```java
/** MarketClock für zeitbasierte Replay-Semantik. */
private final MarketClock marketClock;

/**
 * Geladene LLM-Call-Records für den aktuellen Backtest-Tag, sortiert nach
 * requestMarketTime ascending. Leer wenn kein Replay-Modus aktiv.
 */
private final List<LlmCallRecord> replayRecords;

/** Aktueller Lesezeiger in replayRecords für O(n)-Traversal. */
private int replayPointer;
```

**Neuer Konstruktor:**

```java
/**
 * Creates a CachedAnalyst in replay mode, using pre-loaded LlmCallRecords.
 * The MarketClock is used to determine whether a record's responseMarketTime has been reached.
 *
 * @param marketClock   market clock for latency-aware replay semantics
 */
public CachedAnalyst(MarketClock marketClock) {
    this.marketClock = Objects.requireNonNull(marketClock, "marketClock must not be null");
    this.cache = new ConcurrentHashMap<>();
    this.replayRecords = new ArrayList<>();
    this.replayPointer = 0;
}
```

**`preloadFromRecords()`:**

```java
/**
 * Preloads LLM call records for latency-aware replay.
 * Records must be sorted by requestMarketTime ascending (callers responsibility).
 * Resets the internal replay pointer.
 *
 * @param records list of LLM call records ordered by requestMarketTime ascending
 */
public void preloadFromRecords(List<LlmCallRecord> records) {
    Objects.requireNonNull(records, "records must not be null");
    replayRecords.clear();
    replayRecords.addAll(records);
    replayPointer = 0;
    LOG.info("CachedAnalyst: preloaded {} LLM call records for replay", records.size());
}
```

**Überarbeitete `analyze()`-Methode — Entscheidungsbaum:**

```
analyze(snapshot, context):

    1. Wenn replayRecords nicht leer (REPLAY-Modus):
       → findLatestAvailableRecord(marketClock.now())
         → Iteriere replayRecords von replayPointer aufwärts
         → Finde den letzten Record mit responseMarketTime <= clock.now()
         → Wenn gefunden: deserialize(record.analysis()) → LlmAnalysis zurückgeben
         → Wenn nicht gefunden (Clock ist noch vor dem ersten Record): null zurückgeben

    2. Wenn replayRecords leer (klassischer Hash-Cache-Modus, rückwärtskompatibel):
       → computeCacheKey(snapshot, context)
       → cache.get(key)
       → Wenn cache hit: gecachte Analyse zurückgeben (bestehende Logik)
       → Wenn cache miss: null zurückgeben (NICHT mehr UNCERTAIN-Default!)
```

**Wichtig: replayPointer-Optimierung**

Da `replayRecords` nach `requestMarketTime` sortiert und die SimClock monoton voranschreitet,
kann der `replayPointer` nur vorwärts laufen. Bei jedem `analyze()`-Aufruf:
- Advance `replayPointer` so weit vorwärts wie `responseMarketTime <= clock.now()`
- Der zuletzt gültige Record ist `replayRecords.get(replayPointer - 1)` (oder kein Record wenn `replayPointer == 0`)
- Das ergibt O(n) über alle Bars gesamt, nicht O(n) pro Bar

**Cache-Miss-Verhalten geändert:**

Der alte Code gab bei Cache-Miss ein `UNCERTAIN/0.0`-Objekt zurück.
Der neue Code gibt `null` zurück. Das ist eine semantisch wichtige Änderung:
- `null` bedeutet: "keine LLM-Information verfügbar" → `TradingPipeline` behandelt es als
  "kein Trigger für Failure-Counter"
- UNCERTAIN/0.0 bedeutete: "LLM hat geantwortet, ist aber unsicher" → blockierte Entries
  durch das `regimeConfidence < 0.5`-Gate

### `LlmCallRecordDeserializer` (odin-brain, `de.its.odin.brain.llm`)

```java
/**
 * Deserializes the JSON payload stored in {@link LlmCallRecord#analysis()} back into
 * a {@link LlmAnalysis} record for use by {@link CachedAnalyst} in replay mode.
 *
 * <p>Defensively handles unknown fields and schema drift between record versions.
 * Returns null on critical deserialization failures (missing required fields).
 */
public final class LlmCallRecordDeserializer {

    private static final Logger LOG = LoggerFactory.getLogger(LlmCallRecordDeserializer.class);

    private final ObjectMapper objectMapper;

    /** Creates a deserializer with a default ObjectMapper (unknown fields ignored). */
    public LlmCallRecordDeserializer() {
        this.objectMapper = new ObjectMapper()
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                .registerModule(new JavaTimeModule());
    }

    /**
     * Deserializes the given JSON string to a LlmAnalysis record.
     * Returns null if the JSON is null, blank, or missing required fields.
     *
     * @param analysisJson the JSON string from LlmCallRecord#analysis()
     * @param record       source record (used for metadata fields not in JSON)
     * @return deserialized LlmAnalysis, or null on failure
     */
    @Nullable
    public LlmAnalysis deserialize(String analysisJson, LlmCallRecord record) { ... }
}
```

### `BacktestRunner` — REPLAY-Modus-Erkennung

In `runSingleDay()`, nach dem Laden der Bars und Aufbau des POJO-Graphen:

```java
// Determine replay mode: check if LlmCallRecord entries exist for this backtestId + date
boolean replayAvailable = llmCallRecordRepository != null
        && hasReplayRecordsForDate(batchId, instruments, tradingDate);

if (replayAvailable) {
    LOG.info("REPLAY mode activated for {}: loading existing LLM records", tradingDate);
    initializeReplayAnalyst(cachedAnalyst, batchId, instruments, tradingDate, simClock);
} else {
    LOG.info("REAL_LLM mode for {}: LLM records will be created on first run", tradingDate);
    // bestehende REAL_LLM-Logik (ODIN-055)
}
```

**`hasReplayRecordsForDate()`:**
- Query: `llmCallRecordRepository.existsByBacktestIdAndInstrumentIdAndRequestMarketTimeBetween(...)`
- Zeitbereich: `tradingDate T00:00` bis `tradingDate T23:59` in Exchange-TZ
- Neue Repository-Methode erforderlich (in ODIN-055 ergänzen oder hier hinzufügen)

**`initializeReplayAnalyst()`:**
- Lädt alle Records für `backtestId + instrumentId + tradingDate` (nach `requestMarketTime` sortiert)
- Ruft `cachedAnalyst.preloadFromRecords(records)` auf

### `TradingPipeline` — Unterscheidung Replay vs. Failure

Die `resolveLatestLlmAnalysis()`-Methode (aus ODIN-055 überarbeitet) muss zwischen zwei
null-Rückgaben unterscheiden:

1. `CachedAnalyst.analyze()` gibt null zurück, weil noch kein Record zeitlich verfügbar ist
   → KEIN `degradationManager.onLlmFailure()`. Das ist erwartetes Verhalten.

2. `llmAnalyst.analyze()` wirft eine Exception (echter Fehler)
   → `degradationManager.onLlmFailure()` wie bisher

**Implementierungsoption A — neues Enum:**

```java
// In TradingPipeline (private, nicht im Port):
private enum LlmResolveResult { AVAILABLE, NOT_YET_AVAILABLE, FAILED }
```

Interne Hilfsmethode `resolveLlmInternal()` gibt `LlmResolveResult + LlmAnalysis` zurück.

**Implementierungsoption B — Marker-Exception:**

`CachedAnalyst.analyze()` wirft eine spezielle checked Exception `LlmNotYetAvailableException`
statt `null` zurückzugeben. Die Pipeline fängt diese separat.

**Empfehlung:** Option A (Enum) — kein Exception-as-Control-Flow, klarer Code.
Der Port `LlmAnalyst` bleibt unverändert.

**Konkrete Umsetzung in `resolveLatestLlmAnalysis()`:**

```java
private LlmAnalysis resolveLatestLlmAnalysis(MarketSnapshot snapshot) {
    if (degradationManager.getCurrentMode() == DegradationMode.QUANT_ONLY) {
        return null;
    }

    // Sampling-Policy-Check (aus ODIN-055)
    if (!samplingPolicy.shouldSample(snapshot, currentBarIndex, lastSampleBarIndex)) {
        return lastLlmAnalysis;  // kann null sein — das ist okay
    }

    // LLM-Call via Port
    try {
        LlmAnalysis freshAnalysis = llmAnalyst.analyze(snapshot, runContext);
        // null bedeutet "noch kein Record verfügbar" (CachedAnalyst REPLAY-Modus)
        // — kein Failure, kein DegradationManager-Update
        if (freshAnalysis != null) {
            lastLlmAnalysis = freshAnalysis;
            samplingPolicy.onSampleTaken(currentBarIndex);
            lastSampleBarIndex = currentBarIndex;
            persistLlmCallRecord(freshAnalysis, snapshot);  // ODIN-055
            degradationManager.onLlmSuccess();
        }
        // null-Fall: LLM-Call war technisch erfolgreich, aber noch kein Record zeitlich verfügbar
        // → lastLlmAnalysis unverändert beibehalten (war schon null oder alte Analyse)
        return lastLlmAnalysis;
    } catch (Exception exception) {
        LOG.warn("LLM analysis failed for {}: {} — using stale analysis",
                instrumentId, exception.getMessage(), exception);
        degradationManager.onLlmFailure(...);
        return lastLlmAnalysis;
    }
}
```

### Neue Repository-Methode (odin-brain, `LlmCallRecordRepository`)

```java
/**
 * Checks whether any LLM call records exist for the given backtest batch, instrument,
 * and date range. Used by BacktestRunner to determine REPLAY vs. REAL_LLM mode.
 */
boolean existsByBacktestIdAndInstrumentIdAndRequestMarketTimeBetween(
        String backtestId, String instrumentId,
        Instant from, Instant to);

/**
 * Finds all LLM call records for a backtest batch, instrument, and date range,
 * ordered by requestMarketTime ascending. Used to preload the CachedAnalyst
 * in replay mode.
 */
List<LlmCallRecordEntity> findByBacktestIdAndInstrumentIdAndRequestMarketTimeBetweenOrderByRequestMarketTimeAsc(
        String backtestId, String instrumentId,
        Instant from, Instant to);
```

### Konfiguration

Kein neuer Konfigurationsschalter für REPLAY vs. REAL_LLM. Die Erkennung ist automatisch
(DB-Abfrage vor Tagesstart). Das ist lean und vermeidet manuellen Overhead.

Optional kann ein Property ergänzt werden um den REPLAY-Modus zu erzwingen oder zu deaktivieren:
```properties
# Optional: erzwingt REAL_LLM (ignoriert vorhandene Records), Default: false
odin.backtest.llm.force-real-llm=false
```
Nur ergänzen wenn der Implementierer einen konkreten Use-Case sieht — nicht proaktiv
hinzufügen (Lean-Prinzip).

---

## Konzept-Referenzen

| Dokument | Abschnitt |
|----------|-----------|
| `docs/backend/architecture/05-llm-integration.md` | Kap. 10: CachedAnalyst, LLM-Timing-Modell (Δt_model), `availableFromMarketTime` |
| `docs/backend/architecture/05-llm-integration.md` | Kap. 4: TTL/Freshness-Semantik, Default-Werte bei fehlendem Output |
| `docs/backend/architecture/02-realtime-pipeline.md` | SimClock-Semantik, Bar-Replay vs. Tick-Replay |
| `docs/backend/architecture/06-rules-engine.md` | Kap. 12: Simulation-Semantik, identischer Codepfad |

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
- [ ] JavaDoc auf allen neuen/geänderten public Klassen und Methoden
- [ ] `LlmCallRecordDeserializer` ist defensive (FAIL_ON_UNKNOWN_PROPERTIES = false)
- [ ] `replayPointer`-Optimierung in `CachedAnalyst` korrekt implementiert (O(n) gesamt, nicht O(n) pro Bar)
- [ ] Kein `Instant.now()` im Trading-Codepfad — ausschliesslich `MarketClock.now()`
- [ ] Port `LlmAnalyst` (odin-api) bleibt unverändert — keine Breaking-Change-Risiken

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] `CachedAnalystReplayTest`:
  - `analyze()` gibt `null` wenn Clock vor erstem Record liegt
  - `analyze()` gibt ersten Record zurück wenn `responseMarketTime <= clock.now()`
  - `analyze()` gibt letzten verfügbaren Record zurück wenn Clock zwischen Records liegt
  - `replayPointer`-Fortschritt ist korrekt (kein Reset bei jedem Call)
  - `preloadFromRecords()` mit leerem Input: kein Fehler
- [ ] `LlmCallRecordDeserializerTest`:
  - Korrekte Deserialisierung eines vollständigen JSON-Payloads
  - Unbekannte Felder werden ignoriert (kein Fehler)
  - Fehlendes Pflichtfeld (`regime`): gibt `null` zurück, loggt WARN
  - Null/leerer Input: gibt `null` zurück ohne Exception
- [ ] `TradingPipelineReplayModeTest`:
  - Bei `CachedAnalyst.analyze() == null`: `degradationManager.onLlmFailure()` wird NICHT aufgerufen
  - Bei `llmAnalyst.analyze()` Exception: `degradationManager.onLlmFailure()` wird aufgerufen

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] `BacktestRunnerReplayIntegrationTest` (Zonky Embedded PostgreSQL):
  - Tag 1: REAL_LLM-Modus erkannt (keine Records vorhanden)
  - After Tag 1: Records in DB vorhanden
  - Tag 2 (selbe batchId): REPLAY-Modus erkannt, Records geladen
  - REPLAY-Lauf produziert mindestens so viele Trades wie REAL_LLM-Lauf
- [ ] `LlmCallRecordRepositoryIntegrationTest` (ergänzen):
  - `existsByBacktestIdAndInstrumentIdAndRequestMarketTimeBetween` gibt `true` wenn Records existieren
  - `findByBacktestIdAndInstrumentIdAndRequestMarketTimeBetweenOrderByRequestMarketTimeAsc` liefert korrekte Sortierung

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT nach Edge-Cases gefragt:
  - Was passiert wenn `replayRecords` Records für mehrere Tage enthält (versehentlich)?
  - Was passiert wenn `responseMarketTime` in einem Record fehlt oder Null ist?
  - Ist die `replayPointer`-Optimierung korrekt wenn der SimClock-Stand zwischen zwei
    Record-Timestamps liegt?
  - Kann Look-Ahead-Bias entstehen wenn `responseMarketTime` falsch berechnet wurde (ODIN-055)?
- [ ] Findings bewertet und eingearbeitet
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch ChatGPT (drei Dimensionen)

Da Gemini-Pool dauerhaft ausgefallen ist, deckt ChatGPT alle drei Review-Dimensionen ab.

- [ ] **Dimension 1 — Spec-Review:** Vergleich der überarbeiteten `CachedAnalyst`-Implementierung
  gegen Kap. 5 (LLM-Timing-Modell, Δt_model, `availableFromMarketTime`-Semantik). Ist die
  Implementierung korrekt? Wird Look-Ahead-Bias korrekt verhindert?
- [ ] **Dimension 2 — Code-Quality-Review:** `replayPointer`-Logik, `null`-Safety in
  `LlmCallRecordDeserializer`, Trennung von Failure vs. "not yet available" in `TradingPipeline`,
  Jackson-Konfiguration, JavaDoc-Vollständigkeit.
- [ ] **Dimension 3 — Praxis-Review:** Ist die Auto-Detection von REPLAY vs. REAL_LLM sinnvoll?
  Gibt es Situationen (teilweise vorhandene Records, abgebrochener REAL_LLM-Lauf) bei denen
  das Auto-Detection-Verhalten problematisch ist?
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` in `T:/codebase/its_odin/temp/userstories/ODIN-056_latency-aware-replay/` erstellt
  mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekräftiger Message
- [ ] Push auf Remote
- [ ] Temporäre Protokolldateien gelöscht (Housekeeping-Regel aus MEMORY.md)

---

## Notizen für den Implementierer

### Kritischer Punkt: Zwei CachedAnalyst-Betriebsmodi

Nach dieser Story hat `CachedAnalyst` zwei Modi:

**Modus 1 — Hash-Cache (klassisch, rückwärtskompatibel):**
Wird benutzt wenn `replayRecords` leer ist. `preloadCache(Map<String,LlmAnalysis>)` befüllt
`cache`. Cache-Key ist SHA-256-Hash. Dieser Modus wird weiterhin von Unit-Tests und
möglichen alten Integrations-Szenarien genutzt.

**Modus 2 — Time-based Replay:**
Wird benutzt wenn `preloadFromRecords(List<LlmCallRecord>)` aufgerufen wurde. Records werden
zeitbasiert abgefragt. `cache` (HashMap) wird in diesem Modus nicht verwendet.

Der Implementierer muss sicherstellen dass die beiden Modi sich nicht beeinflussen.
Empfehlung: Im `analyze()`-Einstiegspunkt erst prüfen ob `replayRecords` nicht leer →
Replay-Pfad. Sonst: Hash-Cache-Pfad.

### Kritischer Punkt: Abgebrochene REAL_LLM-Läufe

Wenn ein REAL_LLM-Lauf in der Mitte abgebrochen wird (z.B. nach 2 Stunden Handelstag),
sind nur Records für den ersten Teil des Tages vorhanden. Ein Folgelauf würde den
REPLAY-Modus aktivieren (Records existieren), hätte aber für den zweiten Tagesteil
keine Records. Das Ergebnis: `CachedAnalyst` gibt `null` für den zweiten Tagesteil —
das ist akzeptables Verhalten (LLM-Information fehlt, Entries blockiert).

Der Implementierer soll dieses Szenario in `protocol.md` dokumentieren und ggf. in
ChatGPT-Review diskutieren. Eine automatische Erkennung ("Records decken weniger als X%
des Tages ab → REAL_LLM starten") ist Out of Scope für diese Story, kann aber als
zukünftige Verbesserung notiert werden.

### `analysisJson`-Format

Die in `LlmCallRecord.analysis()` gespeicherten JSON-Daten werden von `LlmCallRecordDeserializer`
deserialisiert. Das Format ist das vollständige Jackson-Serialisierungs-Format von `LlmAnalysis`.

Sicherstellen dass:
1. `LlmAnalysis` vollständig mit Jackson serialisierbar ist (alle Felder, alle Enum-Typen)
2. Jackson-Module für Java-Time (`JavaTimeModule`) registriert sind
3. `Instant`-Felder korrekt serialisiert werden (ISO-8601-Format)

Falls `LlmAnalysis` aktuell nicht direkt serialisierbar ist (z.B. wegen fehlender
`@JsonCreator`-Annotation oder fehlender Default-Konstruktoren), muss ein Zwischen-DTO
eingeführt werden. Das soll der Implementierer beim Lesen der bestehenden `LlmAnalysis`-Record-Definition
prüfen.

### Beziehung zu ODIN-055: `backtestId`

ODIN-055 hat die Frage offen gelassen wie `backtestId` in `LlmCallRecord` befüllt wird
(entweder via `RunContext`-Erweiterung oder direkter Parameter in `TradingPipeline`).
Diese Story setzt voraus dass `backtestId` korrekt gesetzt ist — ohne korrekte `backtestId`
kann der REPLAY-Modus keine Records finden.

Vor Implementierungsbeginn sicherstellen dass ODIN-055 `backtestId` korrekt implementiert hat.
Falls nicht: als blockierenden Befund in `protocol.md` notieren und auf ODIN-055-Fix warten.

### Vorhandene SimClock-Semantik nutzen

`SimClock.now()` gibt bereits den korrekten simulierten Markt-Zeitpunkt zurück, der mit
jedem Bar-Close fortschreitet. Kein Änderungsbedarf. Der `replayPointer`-Mechanismus
in `CachedAnalyst` nutzt diese Eigenschaft direkt.

Die Konsequenz: Wenn SimClock bei einem bestimmten Bar-Close auf `10:05:00` steht und der
erste `LlmCallRecord` `responseMarketTime = 10:04:12` hat, ist der Record ab dem Bar-Close
um 10:05 verfügbar. Das ist die gewünschte Latenz-Simulation.

### Logging und Observability

Im REPLAY-Modus soll beim Start des Tages geloggt werden:
- Anzahl der geladenen Records
- Zeitspanne (erster `requestMarketTime` bis letzter `responseMarketTime`)
- Welche Instrumente betroffen sind

Beim ersten Cache-Hit und beim ersten `null`-Return (kein Record verfügbar) soll DEBUG geloggt werden.
Das hilft beim Debugging von Szenarien wo Entries unerwartet blockiert werden.
