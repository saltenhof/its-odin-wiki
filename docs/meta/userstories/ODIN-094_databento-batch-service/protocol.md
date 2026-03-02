# Protokoll: ODIN-094 — DatabentoBatchService — Download Orchestration

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (25 Tests, alle bestanden)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (7 Tests, alle bestanden)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### 1. DatabentoBatchService als POJO (kein @Service)
Gemaess module-structure.md ist odin-data ein Pro-Pipeline-Modul. Keine Spring-Annotations auf Service-Klassen.
Bean-Erstellung erfolgt in odin-app (LiveWiringConfig/SharedConfig).

### 2. DabentoApi via Factory im Konstruktor erstellt
Der Service erhaelt die DabentoApiFactory (nicht direkt DabentoApi), um die Client-Erstellung korrekt zu kapseln.
Die Factory wird einmal aufgerufen im Konstruktor — der resultierende DabentoApi-Client ist ein Singleton.

### 3. downloadBars() als UnsupportedOperationException
Gemaess Issue Out-of-Scope: "OHLCV-Parsing in die bestehende intraday_bar (Integration mit IntradayBarJdbcRepository)".
Die Methode ist implementiert als explizite UnsupportedOperationException mit klarer Begründung.

### 4. Tageweise Iteration statt Range-Download
Jeder Tag wird einzeln verarbeitet mit eigenem Audit-Log-Eintrag. Bei Fehler an Tag 3 von 5 sind Tag 1 und 2 bereits committed.
Dies entspricht dem Konzeptdokument 003-api-client-design.md Abschnitt 5.2.

### 5. Streaming mit BufferedSource
CSV-Responses werden zeilenweise via OkHttp BufferedSource gelesen, nicht als Ganzes in den RAM geladen.
Die try-with-resources-Kette (Response -> ResponseBody -> BufferedSource) verhindert Connection-Leaks.

### 6. Download-Duration und Persist-Duration sauber getrennt
Persist-Duration wird pro Batch-Aufruf kumuliert (Instant.now() vor/nach persistBatch).
Download-Duration = Gesamtzeit minus Persist-Duration (nach Gemini-Review-Fix).

### 7. Integrationstests mit Stub-Repositories
Da DatabentoBatchServiceIntegrationTest die reale HTTP-Strecke (MockWebServer + OkHttp + Retrofit) und den realen CsvParser testet,
wurden die JDBC-Repositories als In-Memory-Stubs implementiert (Subklassen mit Override). Dies testet die Orchestrierung End-to-End
ohne DB-Abhaengigkeit auf Service-Ebene. Die Repository-IntegrationTests existieren separat (ODIN-093).

### 8. rowCount zaehlt geparste Zeilen, nicht DB-Inserts
Design-Entscheidung: rowCount in DataIngestResult zaehlt die erfolgreich geparsten Zeilen, nicht den Return-Wert
von persistBatch (der bei ON CONFLICT DO NOTHING geringer sein kann). Dies ist bewusst, da rowCount die
Datenmenge widerspiegeln soll, nicht die Uniqueness.

## Offene Punkte
- downloadBars() ist nicht implementiert (Out of Scope gemaess Issue)
- Parallelisierung mehrerer Schemas (Out of Scope gemaess Issue)
- Wochenenden/Feiertage: Service iteriert ueber alle Kalendertage inkl. Nicht-Handelstage.
  Databento gibt leere Responses fuer Nicht-Handelstage zurueck (0 Zeilen, 0 Kosten).
  Ein TradingCalendar-Filter waere eine Optimierung, aber kein Bug. FUTURE-WORK.

## ChatGPT-Sparring

### Session
ChatGPT hat 7 Kategorien von Edge-Cases identifiziert. Die wichtigsten umgesetzten Tests:

**Umgesetzt (7 neue Tests):**
1. `startEqualsEndReturnsZeroNoHttpCalls` — startDate == endDate => no-op
2. `startAfterEndReturnsZeroNoHttpCalls` — startDate > endDate => no-op
3. `costExactlyAtLimitAllowsDownload` — cost == limit darf nicht abbrechen
4. `markFailedThrowsButOriginalExceptionPropagates` — markFailed-Fehler verdeckt nicht Root-Cause
5. `errorMessageTruncatedTo1000Chars` — Laenge-Pruefung fuer DB-Feld
6. `nullResponseBodyMarksCompletedWithZero` — null body Handling
7. `rowCountCountsParsedLinesNotInserted` — Metrik-Definition validiert

**Bewertet und verworfen:**
- Concurrency-Tests: Der Service ist nicht thread-safe by design. DB-Unique-Constraints schuetzen auf DB-Ebene.
  Explizite Concurrency-Tests sind nicht sinnvoll auf Unit-Ebene.
- Riesige CSV-Zeilen: OkHttp/Okio-Puffer-Verhalten ist nicht im Scope dieses Service-Tests.
- Header-Validation: Parser-Verantwortung (ODIN-092), nicht Service-Verantwortung.

## Gemini-Review

### Dimension 1: Code-Bugs
**6 Findings, 4 eingearbeitet:**
1. CRITICAL (Partial Inserts bei Race): NOT APPLICABLE — Repos nutzen ON CONFLICT DO NOTHING, existsIngest prueft alle Eintraege.
2. MAJOR (costUsd nicht in markCompleted): DOKUMENTIERT als Design-Entscheidung — costUsd ist ein Estimate, kein Final-Wert.
3. MAJOR (Resource Leak bei Error Response): FIXED — closeResponseBodyQuietly() vor Exception-Throw hinzugefuegt.
4. MINOR (Ternary Typo raw_symbol/raw_symbol): FIXED — Redundanten Ternary entfernt, direkt "raw_symbol" verwendet.
5. MINOR (downloadDuration inkludiert persistDuration): FIXED — totalElapsed.minus(persistDuration) fuer reine Download-Zeit.
6. MINOR (null getMessage bei NPE): FIXED — Fallback auf exception.getClass().getSimpleName().

### Dimension 2: Konzepttreue
**Alle Akzeptanzkriterien als MATCH bewertet:**
- 5-Schritt-Workflow: MATCH (Step 5 Kompression ist optional im Konzept)
- Idempotenz: MATCH
- Kostenschaetzung + Limit: MATCH
- Streaming + Batch-Persist: MATCH
- Mehrtaegiger Download: MATCH
- Metriken in DataIngestResult: MATCH
- Fehlerbehandlung: MATCH
- downloadBars als UnsupportedOperationException: DEVIATION — begruendet (Out of Scope)

### Dimension 3: Praxis-Gaps
**8 Findings bewertet:**
1. MAJOR (Wochenenden): Kein Bug, Databento gibt leere Response. FUTURE-WORK (TradingCalendar).
2. MINOR (Grosse Downloads): Read-Timeout 300s konfiguriert in DabentoProperties. OK.
3. MAJOR (Schema-Aenderungen): Parser-Verantwortung (ODIN-092). NOT IN SCOPE.
4. CRITICAL (Partial Inserts): NOT APPLICABLE — ON CONFLICT DO NOTHING in allen Repos.
5. MAJOR (Concurrency): DB-Unique-Constraint existiert (ODIN-093). Schuetzt auf DB-Ebene.
6. MAJOR (Timezone): Databento akzeptiert YYYY-MM-DD und interpretiert relativ zum Dataset. OK.
7. MINOR (Speicher voll): Infrastrukturelles Problem. Cleanup bei Re-Run dank ON CONFLICT.
8. MAJOR (Retry): DabentoRetryInterceptor in OkHttp-Stack (ODIN-091) handled 429/5xx. OK.
