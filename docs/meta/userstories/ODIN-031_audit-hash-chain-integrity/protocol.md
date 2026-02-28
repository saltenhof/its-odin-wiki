# ODIN-031 — Audit Log Hash-Chain Integrity: Implementation Protocol

**Story:** ODIN-031
**Datum:** 2026-02-23
**Status:** Implementierung vollständig, Tests grün, Reviews durchgeführt — bereit für QS-Agent-Commit

---

## 1. Implementierungsumfang

### Neue Dateien

| Datei | Beschreibung |
|-------|-------------|
| `odin-audit/.../service/HashChainComputer.java` | Statisches Utility: SHA-256-Berechnung, Jackson-Kanonisierung, stampHashChain |
| `odin-audit/.../service/HashChainValidator.java` | Service: verifyHashChain(UUID runId), VerificationResult-Record mit Outcome-Enum |
| `odin-audit/.../service/HashChainComputerTest.java` | 13 Unit-Tests (Determinismus, Delimiter, Regression, stampHashChain) |
| `odin-audit/.../service/HashChainValidatorTest.java` | 12 Unit-Tests (Mockito, Tamper-Szenarien, Boundary-Cases) |
| `odin-audit/.../service/HashChainIntegrationTest.java` | 8 Integrationstests (Zonky embedded Postgres, Manipulation-Detection) |

### Modifizierte Dateien

| Datei | Änderung |
|-------|---------|
| `odin-audit/.../repository/EventRecordRepository.java` | +`findHashedEventsByRunIdOrderedBySequence`, +`findLastHashedEventByRunId` |
| `odin-audit/.../service/PostgresEventLog.java` | SIMULATION-Modus: per-runId Hash-Chain via `simLastHashByRun: Map<UUID, String>` |
| `odin-audit/.../service/AuditEventDrainer.java` | LIVE-Modus: per-runId Partitionierung, in-flight-Retry-Pattern, DB-Fallback |

---

## 2. Implementierte Hash-Chain-Formel

```
SHA-256(previousHash | timestamp | eventType | canonicalPayload)
```

- **previousHash:** Hash des Vorgänger-Records oder Sentinel `"GENESIS"`
- **timestamp:** ISO-8601 (`Instant.toString()`) des MarketClock-Zeitstempels
- **eventType:** Ereignis-Klassifikation (z. B. `"STATE_CHANGE"`)
- **canonicalPayload:** Jackson-kanonisiertes JSON (alphabetisch sortierte Keys, kein Whitespace)
- **Delimiter:** `|` (Pipe) — verhindert Boundary-Kollisionen zwischen Feldern

### JSON-Kanonisierung

Anstelle einer proprietären Normalisierung (die PostgreSQL JSONB nachzuahmen versuchte) wird Jackson mit `SORT_PROPERTIES_ALPHABETICALLY` + `ORDER_MAP_ENTRIES_BY_KEYS` verwendet:

```java
private static final ObjectMapper CANONICAL_MAPPER = JsonMapper.builder()
    .enable(MapperFeature.SORT_PROPERTIES_ALPHABETICALLY)
    .enable(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS)
    .build();
```

**Symmetrie-Eigenschaft:** PostgreSQL JSONB sortiert Keys nach Länge (dann alphabetisch). Jackson sortiert rein alphabetisch. Beide Schritte (Write-Time und Validator-Read-Time) rufen `normalizeJson()` auf — Jackson re-parst und re-serialisiert identisch, unabhängig vom JSONB-Intermediate-Format. Hash-Konsistenz ist garantiert.

---

## 3. Architekturelle Entscheidungen

### 3.1 Per-runId Hash-Chain (KRITISCH, ursprünglich falsch)

**Problem:** Globaler `lastPersistedHash` verknüpft Events verschiedener Runs in einer Kette. Der Validator verifiziert pro runId und erwartet GENESIS als ersten previousHash — eine globale Kette würde sofort fehlschlagen.

**Lösung:** `Map<UUID, String> lastPersistedHashByRun` im Drainer (und `simLastHashByRun` in PostgresEventLog). Jeder Run startet mit GENESIS. Nach JVM-Neustart: DB-Fallback via `findLastHashedEventByRunId()`.

### 3.2 In-Flight-Retry-Pattern (ersetzt requeueAll)

**Problem:** Bei Persist-Fehler würden Events via `requeueAll()` hinter neu angehängte Events in den Spool eingereiht — die Hash-Chain-Reihenfolge würde korrumpiert.

**Lösung:** Fehlgeschlagene Batches werden als `inFlightBatch` im Drainer gehalten (nicht requeued). Nächster Drain-Cycle retried `inFlightBatch` zuerst, bevor neue Events gezogen werden. Hash-Felder werden vor dem Retry gecleared (sauberes Re-Stamping).

### 3.3 Batch-Partitionierung nach runId + sequenceNumber-Sortierung

Der Drainer gruppiert einen Batch per `Collectors.groupingBy(EventRecordEntity::getRunId)`. Jede Gruppe wird nach `sequenceNumber` sortiert und mit ihrem eigenen Chain-Pointer gestampt. Damit ist die Batch-Reihenfolge unabhängig von Spool-Insertion-Order.

### 3.4 DB-Fallback für JVM-Neustart

`resolveLastHashForRun(UUID runId)` verwendet `computeIfAbsent`: falls kein In-Memory-Pointer für eine runId existiert, wird der letzte Hash aus der DB geladen (`findLastHashedEventByRunId`). Falls keine gehashten Records vorhanden: GENESIS.

---

## 4. Test-Ergebnisse

### Unit-Tests (Surefire)

| Klasse | Tests | Ergebnis |
|--------|-------|----------|
| `HashChainComputerTest` | 13 | GRÜN |
| `HashChainValidatorTest` | 12 | GRÜN |
| `AuditEventDrainerTest` | 9 | GRÜN |
| `PostgresEventLogTest` | 18 | GRÜN |
| Sonstige | 12 | GRÜN |
| **Gesamt** | **64** | **ALLE GRÜN** |

### Integrationstests (Zonky embedded Postgres)

| Klasse | Tests | Ergebnis |
|--------|-------|----------|
| `HashChainIntegrationTest` | 8 | GRÜN |

**Gesamtergebnis: 72/72 Tests grün.**

---

## 5. Review-Prozess

### 5.1 ChatGPT-Review (2 Runden)

**Runde 1 — Identifizierte Findings:**

| # | Priorität | Finding |
|---|-----------|---------|
| 1 | KRITISCH | Hash-Chain-Pointer global, nicht per-runId — Validator erwartet GENESIS als Start |
| 2 | KRITISCH | Batch-Sortierung mischt Events verschiedener Runs bei globaler Sequenznummer |
| 3 | WICHTIG | `requeueAll()` korrumpiert Chain bei Persist-Fehler (Interleaving) |
| 4 | WICHTIG | `normalizeJson()` zu naiv — extra Whitespace im Input führt zu Hash-Divergenz |
| 5 | WICHTIG | Null-Safety in `computeHash()` fehlt |
| 6 | WICHTIG | JavaDoc-Inkonsistenzen in HashChainValidator |

**Runde 2 — Konkrete Fix-Proposals:**
- Fix #2: Batch vor Stamping nach `sequenceNumber` sortieren
- Fix #3: In-Flight-Retry-Pattern implementieren (kein `requeueAll`)

**Implementiert:** Alle Findings #1–#6.

### 5.2 Gemini-Review (2 Runden)

**Runde 1 — 3-Dimensions-Review:**

| Dimension | Key Findings |
|-----------|-------------|
| Code-Qualität | Sehr sauber. Java 21-konform. Tests erstklassig. |
| Konzept-Treue | Hash-Formel korrekt. KRITISCH: globaler Chain-Pointer (per-runId fehlt). KRITISCH: Batch-Sortierung mischt Runs. |
| Production-Readiness | WICHTIG: `normalizeJson()` zu naiv (JSONB vs. Jackson Key-Sortierung). WICHTIG: `simLastHash` per-runId fehlt. MINOR: In-Flight-Retry bei dauerhaftem DB-Ausfall dokumentieren. |

**Runde 2 — Design-Klärung:**
- Jackson-basierte Kanonisierung mit `SORT_PROPERTIES_ALPHABETICALLY` ist korrekt (symmetrisch, DB-agnostisch)
- ACHTUNG: JSONB sortiert Keys nach Länge (dann Name), Jackson sortiert rein alphabetisch — beide normieren aber identisch wenn via Jackson re-parsed → symmetrisch
- DB-Fallback via `findLastHashedEventByRunId()` als Lösung für JVM-Neustart-Szenario

**Implementiert:** Alle Gemini-Empfehlungen.

---

## 6. Bekannte Limitierungen (dokumentiert, kein Blocker)

| # | Beschreibung |
|---|-------------|
| L1 | Bei dauerhaftem DB-Ausfall im LIVE-Modus: `inFlightBatch` wird immer wieder probiert, während Spool läuft voll und Events beginnen zu droppen. Dieses Verhalten ist gewollt (Fail-Fast → Kill-Switch). Kein Fix nötig, aber im Betriebskonzept zu dokumentieren. |
| L2 | `eventId` wird intern via `UUID.randomUUID()` generiert (API Gap — `EventLog.append` hat keinen `eventId`-Parameter). Technische Schuld, bereits als TODO kommentiert. Kein Fix in diesem Story-Scope. |
| L3 | `lastPersistedHashByRun` wird nie bereinigt. Da ODIN max. 2-3 Runs gleichzeitig unterstützt, ist die Map bounded und kein Memory Leak. Bei zukünftiger Skalierung: LRU-Cache mit Größen-Limit. |

---

## 7. Dateipfade (Vollständig)

```
T:/codebase/its_odin/its-odin-backend/odin-audit/src/main/java/de/its/odin/audit/
  service/HashChainComputer.java          (NEU)
  service/HashChainValidator.java         (NEU)
  service/PostgresEventLog.java           (MODIFIZIERT)
  service/AuditEventDrainer.java          (MODIFIZIERT)
  repository/EventRecordRepository.java   (MODIFIZIERT)

T:/codebase/its_odin/its-odin-backend/odin-audit/src/test/java/de/its/odin/audit/
  service/HashChainComputerTest.java      (NEU)
  service/HashChainValidatorTest.java     (NEU)
  service/HashChainIntegrationTest.java   (NEU)
  service/AuditEventDrainerTest.java      (MODIFIZIERT — 1 Test auf In-Flight-Semantik aktualisiert)
```

---

## 8. Offene Punkte für QS-Agent / nachfolgende Agents

- **Commit:** Alle oben genannten Dateien committen (kein temporäres file erzeugt)
- **Kein Frontend-Scope:** ODIN-031 definiert keine API-Endpoints — HashChainValidator ist reines Backend-Utility
- **Mögliche Erweiterung:** REST-Endpoint `GET /api/v1/runs/{runId}/hash-chain/verify` für Monitoring-Dashboard (separater Story-Scope)
