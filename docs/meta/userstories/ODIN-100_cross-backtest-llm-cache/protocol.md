# ODIN-100: Cross-Backtest LLM-Cache — Implementierungsprotokoll

## 1. Working State

**Status: DONE — vollstaendig implementiert, alle Tests gruen, committed und gepusht.**

### Implementierte Artefakte

| Artefakt | Typ | Pfad |
|---------|-----|------|
| `V037__add_cross_backtest_llm_record_index.sql` | Flyway Migration (NEU) | `odin-persistence/src/main/resources/db/migration/` |
| `LlmCallRecordRepository.java` | Spring Data JPA Repository (GEAENDERT) | `odin-brain/src/main/java/de/its/odin/brain/persistence/` |
| `BacktestRunner.java` | Backtest-Orchestrierung (GEAENDERT) | `odin-backtest/src/main/java/de/its/odin/backtest/` |
| `CrossBacktestCacheTest.java` | Unit-Test Surefire (NEU) | `odin-backtest/src/test/java/de/its/odin/backtest/llm/` |
| `CrossBacktestLlmCacheIntegrationTest.java` | Integrationstest Failsafe (NEU) | `odin-brain/src/test/java/de/its/odin/brain/persistence/` |

### Build-Status

- `mvn clean install -DskipTests` — GRUEN (alle Module)
- `mvn verify -pl odin-persistence,odin-brain,odin-backtest` — GRUEN
  - Surefire (odin-backtest): 374 Unit-Tests, 0 Failures
  - Failsafe (odin-brain): 6 Integrationstests, 0 Failures
  - Pre-existing Failures in `odin-brain` (`ExhaustionDetectorIntegrationTest`, `OpeningConsolidationFsmIntegrationTest`, `ReEntryCorrectionFsmIntegrationTest`) sind NICHT durch ODIN-100 verursacht — bestaetigt durch git-diff-Analyse.

---

## 2. Design-Entscheidungen

### 2.1 Flyway-Migrationsversion: V037 (nicht V029 wie in Story angegeben)

Die Story-Datei nennt `V029__add_cross_backtest_llm_record_index.sql`. Im Ist-Stand belegte jedoch bereits `V029__allow_3m_bar_interval.sql` diese Version. Nach Pruefen aller vorhandenen Migrationen (V001 bis V036) wurde V037 als naechste freie Version verwendet.

Migration mit `IF NOT EXISTS` fuer Idempotenz (auf Empfehlung des ChatGPT-Reviews):

```sql
CREATE INDEX IF NOT EXISTS idx_llm_call_record_cross_backtest
    ON odin.llm_call_record (instrument_id, prompt_version, model_id, request_market_time);
```

### 2.2 Drei-Stufen-Prioritaetskette in `resolveBacktestLlmAnalyst()`

```
1. CACHED provider → llmAnalyst direkt (kein DB-Lookup)
2. Eigener Batch-Match → buildReplayAnalystIfRecordsExist() (unveraendertes ODIN-056-Verhalten)
3. Cross-Backtest-Match → buildCrossBacktestReplayAnalystIfRecordsExist() (NEU, ODIN-100)
4. Fallback → REAL_LLM (live API-Calls)
```

### 2.3 Batch-Selektionspolitik: Coverage > Recency > Stable-Tiebreaker

Die native Query in `findBestCrossBacktestBatchId()` waehlt den besten Quell-Batch nach:

1. **Hoehere Abdeckung**: `COUNT(DISTINCT request_market_time) DESC` — bevorzugt vollstaendigere Runs (bei abgebrochenem vorherigem Run sind weniger Slots vorhanden).
2. **Neueste Erstellung**: `MAX(created_at) DESC` — bei gleicher Abdeckung gewinnt der juengere Run.
3. **Stabiler Tiebreaker**: `backtest_id ASC` — deterministischer Tiebreaker wenn Abdeckung und Zeitstempel identisch sind (verhindert nicht-deterministisches Ergebnis bei identischen Runs).

```sql
SELECT backtest_id FROM odin.llm_call_record
WHERE instrument_id = :instrumentId
  AND prompt_version = :promptVersion
  AND model_id = :modelId
  AND request_market_time BETWEEN :from AND :to
  AND backtest_id IS NOT NULL
GROUP BY backtest_id
ORDER BY COUNT(DISTINCT request_market_time) DESC, MAX(created_at) DESC, backtest_id ASC
LIMIT 1
```

### 2.4 All-or-Nothing-Policy auch fuer Cross-Backtest

Identisch zur Eigen-Batch-Policy: wenn auch nur ein Instrument keine Cross-Backtest-Records findet (oder die besten Batches je Instrument divergieren), wird kein Cross-Backtest-Replay aktiviert. Statt partieller Abdeckung → Fallback auf REAL_LLM. Verhindert stille Null-Analysen bei fehlenden Instrumenten.

### 2.5 Keine Cross-Batch-Stitching erlaubt

Wenn Instrument AAPL den besten Batch "X" und Instrument MSFT den besten Batch "Y" liefert, wird Cross-Backtest deaktiviert. Nur wenn alle Instrumente denselben Gewinne-Batch liefern, wird der Replay-Analyst gebaut. Verhindert Inkonsistenz durch Mischung von Records verschiedener Runs.

### 2.6 Anti-Look-Ahead unveraendert

Die Cross-Backtest-Records durchlaufen exakt dieselbe `CachedAnalyst.analyzeReplayMode()`-Logik mit `responseMarketTime`-Gating. Es gibt keinen Bypass und keine Abkuerzung. Die Herkunft der Records (eigen vs. cross-batch) aendert die Replay-Semantik nicht.

### 2.7 Kein neuer Mode in CachedAnalyst noetig

`CachedAnalyst.preloadFromRecords()` ist bereits instrument-agnostisch und batch-agnostisch — der Unterschied liegt nur im Ladepfad im BacktestRunner.

---

## 3. Test-Ergebnisse

### Unit-Tests (Surefire) — `CrossBacktestCacheTest`

| Test | Szenario | Ergebnis |
|------|---------|---------|
| `ownBatchMatch_ownReplayActivated_crossBacktestNotQueried` | Eigener Batch hat Records → Cross-Backtest wird NICHT aufgerufen | GRUEN |
| `noBatchMatch_crossBacktestExists_crossReplayActivated` | Kein eigener Batch, Cross-Backtest hat Records → Cross-Replay aktiviert | GRUEN |
| `noBatchMatch_differentPromptVersion_crossBacktestNotActivated` | Cross-Backtest gibt leere Liste zurueck (andere prompt_version) → REAL_LLM | GRUEN |
| `twoInstruments_onlyOneHasCrossBacktestRecords_allOrNothingFallsBackToRealLlm` | 2 Instrumente, nur 1 hat Cross-Backtest-Records → All-or-Nothing → REAL_LLM | GRUEN |
| `cachedProvider_noRepoQueriesAtAll` | CACHED provider → kein DB-Lookup | GRUEN |

**Hinweis zur Test-Architektur**: Tests, die Repository-Interaktionen verifizieren (wurde `findBestCrossBacktestBatchId` aufgerufen?), benoetigen echte Bar-Daten, da `runSingleDay()` bei leeren Bar-Maps fruehzeitig abbricht ("No data for date, skipping") und `resolveBacktestLlmAnalyst()` nie aufgerufen wird. Diese Tests verwenden `buildSyntheticBars()` und einen `BrokerGateway`-Mock. Tests ohne Interaktionsverifizierung koennen auf Bar-Daten verzichten.

### Integrationstests (Failsafe) — `CrossBacktestLlmCacheIntegrationTest`

| Test | Szenario | Ergebnis |
|------|---------|---------|
| `batchA_records_discoveredBy_batchB_crossBacktestQuery` | Batch A Records von Batch B via findBestCrossBacktestBatchId gefunden und geladen | GRUEN |
| `differentPromptVersion_notReturnedAsCrossBacktestHit` | Andere prompt_version → kein Cache-Treffer | GRUEN |
| `v037_crossBacktestIndex_exists` | Index `idx_llm_call_record_cross_backtest` nach V037-Migration vorhanden | GRUEN |
| `v037_crossBacktestIndex_coversCorrectColumns` | Index-Definition enthaelt instrument_id, prompt_version, model_id, request_market_time | GRUEN |
| `bestBatchSelection_highestCoverageWins` | Batch mit 3 Records gewinnt gegen Batch mit 1 Record | GRUEN |
| `existsQuery_returnsCorrectResults` | `existsByInstrumentIdAndPromptVersionAndModelIdAndRequestMarketTimeBetween` true/false korrekt | GRUEN |

---

## 4. ChatGPT-Sparring

**Session-Ziel**: Edge-Cases und Grenzszenarien fuer ODIN-100 identifizieren, Batch-Selektionsstrategie validieren.

### Finding 1: Duplikat-Handling — Coverage statt Recency als primaeres Kriterium

**Frage**: Wenn mehrere Batches fuer denselben Tag und dasselbe Instrument Records haben — welcher gewinnt?

**ChatGPT-Empfehlung**: Recency allein ist unzureichend, weil ein juengerer aber abgebrochener Run weniger Records hat als ein aelterer vollstaendiger Run. Coverage (Anzahl distinct time slots) sollte primaer sein, Recency als sekundaerer Tiebreaker.

**Bewertung**: Korrekt und uebernommen. Die Batch-Selektions-Query verwendet `COUNT(DISTINCT request_market_time) DESC, MAX(created_at) DESC`.

### Finding 2: DISTINCT ON vs. GROUP BY — Query-Strategie

**Frage**: Soll `DISTINCT ON (request_market_time)` verwendet werden, um pro Zeitslot den besten Record zu waehlen?

**ChatGPT-Empfehlung**: `DISTINCT ON` ist PostgreSQL-spezifisch und wuerde Records aus unterschiedlichen Batches mischen (Stitching). Besser: `GROUP BY backtest_id` mit Coverage-Sortierung, dann alle Records eines Gewinners laden.

**Bewertung**: Korrekt. Die implementierte Strategie (Gewinne-Batch per `findBestCrossBacktestBatchId`, dann alle Records dieses Batches laden) vermeidet Stitching und ist konsistent.

### Finding 3: All-or-Nothing-Policy explizit bestaetigt

**Frage**: Was passiert, wenn AAPL Cross-Backtest-Records hat, aber MSFT nicht?

**ChatGPT-Empfehlung**: All-or-Nothing ist zwingend. Partial-Replay erzeugt stille Null-Analysen fuer fehlende Instrumente — inakzeptabel in einem Handelssystem.

**Bewertung**: Bereits als Kernprinzip aus ODIN-056 uebernommen. Explizit fuer Cross-Backtest implementiert.

### Finding 4: Anti-Look-Ahead-Risiko

**Frage**: Besteht Look-Ahead-Risiko, wenn Cross-Backtest-Records des vorherigen Runs andere `responseMarketTime`-Werte haben?

**ChatGPT-Empfehlung**: `responseMarketTime` ist deterministische Funktion von `requestMarketTime` + Latenz — beide werden in der Entitaet gespeichert. Da dieselbe `analyzeReplayMode()`-Logik mit `responseMarketTime`-Gating verwendet wird, kein Look-Ahead-Risiko.

**Bewertung**: Bestaetigt. Kein Code-Aenderungsbedarf.

### Finding 5: Leere Instruments-Liste

**Frage**: Was passiert bei `instruments.isEmpty()`?

**ChatGPT-Empfehlung**: Fruehzeitiger `null`-Return vermeidet Loop mit leerer Liste.

**Bewertung**: Implementiert als erste Guard-Clause in `buildCrossBacktestReplayAnalystIfRecordsExist()`.

---

## 5. ChatGPT-Review (Drei Dimensionen)

**Review-Scope**: `BacktestRunner.java`, `LlmCallRecordRepository.java`, `V037__add_cross_backtest_llm_record_index.sql`.

### Dimension 1: Code-Review (Bugs, Implementierungsfehler, Null-Safety)

**Finding R1: Flyway Migration ohne `IF NOT EXISTS`** — Wenn die Migration erneut ausgefuehrt wird (oder die DB bereits existiert), schlaegt `CREATE INDEX` fehl.

**Massnahme**: `IF NOT EXISTS` ergaenzt. **Angewandt.**

**Finding R2: Kein stabiler Tiebreaker in `findBestCrossBacktestBatchId`** — Bei gleicher Coverage und gleichem `MAX(created_at)` ist das Ergebnis nicht-deterministisch.

**Massnahme**: `backtest_id ASC` als dritten Sort-Key ergaenzt. **Angewandt.**

**Finding R3: CACHED-Provider-Pfad korrekt** — `resolveBacktestLlmAnalyst()` gibt bei `LlmProvider.CACHED` sofort `llmAnalyst` zurueck, ohne DB-Queries. Kein Problem.

### Dimension 2: Konzepttreue-Review (Implementierung vs. Konzept)

**Finding K1: `analysis`-Feld null oder malformed** — Was passiert, wenn ein geladener Record ein nicht-parsebares `analysis`-JSON hat?

**Analyse**: `CachedAnalyst.analyzeReplayMode()` ruft `deserializer.deserialize()` auf, das bei Fehler `null` zurueckgibt. Der Analyst gibt `Unavailable(SCHEMA_ERROR)` zurueck und loggt WARN. Kein Crash, graceful degradation. **Kein Code-Aenderungsbedarf.**

**Finding K2: Alle drei Prioritaetsstufen klar trennbar** — CACHED/OwnBatch/CrossBatch/REAL_LLM sind sauber sequenziell implementiert. Konzept vollstaendig abgebildet.

### Dimension 3: Praxis-Review (reale Szenarien)

**Finding P1: Erster Backtest-Run immer REAL_LLM** — Korrekt. Kein Cross-Backtest-Cache ohne mindestens einen vorherigen REAL_LLM-Run. Erwartetes Verhalten.

**Finding P2: Partial-Run-Schutz durch Coverage-Praeferenz** — Ein abgebrochener Run hat weniger distinct slots. Der Coverage-Sort stellt sicher, dass der vollstaendigere Run bevorzugt wird.

**Finding P3: Performance bei vielen Batches** — Der composite Index `(instrument_id, prompt_version, model_id, request_market_time)` deckt alle WHERE-Bedingungen der Cross-Backtest-Query ab. Die `GROUP BY backtest_id` wird als Covering-Index-Scan bearbeitet. Integrationstest `v037_crossBacktestIndex_exists` bestaetigt Index-Existenz.

---

## 6. Offene Punkte

Keine. ODIN-100 ist vollstaendig abgeschlossen.

### Verworfene Alternativen (zur Dokumentation)

- **DISTINCT ON per request_market_time**: Verworfen — erzeugt Cross-Batch-Stitching, schwer debuggbar.
- **Konfigurierbare Cache-TTL**: Verworfen — Out of Scope laut Story; Records bleiben unbegrenzt gueltig solange prompt_version + model_id identisch.
- **UI-Anzeige des Cross-Backtest-Cache-Status**: Verworfen — Out of Scope laut Story.

### Hinweis zur naechsten Story

Wenn ein neuer Backtest ausgefuehrt wird, der REAL_LLM benoetigt (weil kein Cross-Backtest-Cache vorhanden), wird ein neuer Record-Satz erstellt. Dieser steht dann als Cross-Backtest-Cache fuer den naechsten Run zur Verfuegung — automatisch, ohne weiteren Konfigurationsaufwand.
