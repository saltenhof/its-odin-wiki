# Protokoll: ODIN-095 — Databento Download REST API (DatabentoController)

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (27 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (22 Tests)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push (Runde 1)
- [x] QA-Remediation Runde 2: 3 Findings behoben
- [x] Remediation-Review ChatGPT + 3x Gemini
- [x] Commit & Push (Runde 2)

## Design-Entscheidungen

### Controller-Architektur
- Controller injiziert `HistoricalDataProvider` (Port-Interface) fuer Download/Kostenschaetzung und `DataIngestLogJdbcRepository` direkt fuer Read-Only Ingest-Log-Abfragen.
- Schema-Routing via switch-Expression: MBP_1 -> downloadQuotes, TRADES -> downloadTrades, OHLCV_* -> downloadBars mit dynamischer Bar-Interval-Aufloesung.
- `@Validated` auf Controller-Ebene fuer Bean Validation der Request-DTOs und Query-Parameter.

### Synchroner Download
- Bewusste Entscheidung: Downloads sind synchron, Client wartet. Fuer manuelles Triggering aus der Desktop-Anwendung akzeptabel (siehe GitHub Issue Notizen). Async/Job-Pattern waere Overengineering fuer den aktuellen Use Case.

### Ingest-Log Filter-Logik
- Vier Kombinationen: (1) symbol + date range, (2) symbol only, (3) date range only, (4) alle.
- Gemini Dimension 1 hat korrekt identifiziert, dass die urspruengliche Implementierung den Fall "nur Datumsbereich ohne Symbol" nicht abdeckte. Behoben durch Hinzufuegen von `findByDateRange()`.

### Error Handling
- Databento-spezifische Exceptions werden im GlobalExceptionHandler behandelt (DatabentoCostLimitException -> 422, DabentoAuthException -> 502, DabentoApiException -> 502).
- Bestehende Handler (IllegalArgumentException -> 400, ConstraintViolationException -> 400) decken Validierungsfehler ab.

### DTOs
- Alle DTOs als Records implementiert (DatabentoDownloadRequest, DatabentoDownloadResponse, DatabentoCostEstimateResponse, DatabentoIngestLogResponse).
- Response-DTOs enthalten Schema als API-String (z.B. "mbp-1") statt Enum-Name fuer API-Konsistenz.

### Repository-Erweiterungen
- `DataIngestLogJdbcRepository` um Abfrage-Methoden erweitert: `findBySymbol()`, `findBySymbolAndDateRange()`, `findByDateRange()`, `findAll()`.
- RowMapper defensiv gegen NULL-Werte in `started_at` (obwohl NOT NULL in Schema).

## Offene Punkte

### Hardcoded Dataset (XNAS.ITCH)
Der Cost-Estimate-Endpoint verwendet hartcodiertes Dataset "XNAS.ITCH". Fuer Futures/Optionen muesste das parametrisierbar werden. Aktuell nur NASDAQ-Aktien relevant. Kein unmittelbarer Handlungsbedarf.

### Async Download Pattern (Zukunft)
Fuer groessere Deployments waere ein asynchrones Trigger-&-Poll-Pattern (HTTP 202 + Job-ID) sinnvoll. Aktuell nicht noetig (Desktop-Anwendung, 1-2 gleichzeitige Downloads).

## ChatGPT-Sparring

### Vorgeschlagene Testszenarien (20 Kategorien)
ChatGPT hat umfangreiche Edge-Cases vorgeschlagen. Bewertung und Umsetzung:

**Umgesetzt:**
1. Explizite JSON-Nulls vs. fehlende Felder (Integration-Test)
2. Invalider Enum-Wert im Schema (Integration-Test)
3. Leerer Body (Integration-Test)
4. Unbekannte JSON-Felder — verifiziert, dass Jackson im Record-Modus diese ablehnt (Integration-Test)
5. Unerwartete RuntimeException -> 500 (Unit + Integration-Test)
6. NULL-Felder in JSON (Integration-Test)
7. Falsche HTTP-Methode auf Endpoints -> 405 (Integration-Test)
8. Exakt 31 Tage Grenze (Unit-Test)
9. Ingest-Log mit nur startDate/endDate ohne Symbol (Unit-Test)
10. Error Propagation Tests (Unit-Test)

**Verworfen (begruendet):**
- Cross-month/leap-day Boundaries: ChronoUnit.DAYS.between() handhabt diese korrekt
- Concurrent parallel requests: Controller ist stateless, keine shared mutable state
- Content negotiation (415/406): Standard Spring Boot Verhalten, kein projektspezifischer Test noetig
- Large numeric values: Jackson Standard-Serialisierung ausreichend

## Gemini-Review

### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
**Findings:**
1. (HIGH) Filter-Logik in getIngestLog ignorierte Datumsbereich ohne Symbol -> **BEHOBEN**: `findByDateRange()` hinzugefuegt, if/else-Kette erweitert.
2. (MEDIUM) NullPointerException-Risiko im RowMapper bei started_at -> **BEHOBEN**: Defensiver null-Check hinzugefuegt.
3. (LOW) Kein Date-Validation auf ingest-log Endpoint -> **AKZEPTIERT**: Ingest-Log ist nur eine Abfrage, keine Gefahr bei umgekehrter Reihenfolge.
4. (INFO) Synchroner Download = Timeout-Risiko -> **AKZEPTIERT**: Dokumentierte Design-Entscheidung per Issue-Spec.

### Dimension 2: Konzepttreue-Review
**Findings:**
- AC1-AC7: Alle erfuellt
- AC8 (OpenAPI): Nicht erfuellt — aber projektweites Pattern-Gap (kein Controller hat OpenAPI-Annotations)
- Technische Requirements: Alle erfuellt (Port-Injection, Schema-Routing, @Validated, Records, JavaDoc)

### Dimension 3: Praxis-Review
**Findings:**
1. HTTP Timeouts bei langen Downloads -> AKZEPTIERT (Design-Entscheidung, manuelles Desktop-Triggering)
2. Hardcoded Dataset/Venue -> Notiert als Offener Punkt fuer Zukunft
3. Timezone-Ambiguitaet -> Databento API nutzt UTC, LocalDate funktioniert korrekt
4. Fehlende Correlation-IDs im Logging -> Guter Punkt fuer zukuenftige Observability-Story
5. Thread-Pool-Exhaustion -> Bekanntes Risiko bei synchronem Design, akzeptabel fuer 1-2 gleichzeitige Downloads
6. Idempotency -> Wird vom DatabentoBatchService ueber existsIngest() gehandhabt, nicht Controller-Ebene

## QA-Remediation Runde 2

### Behobene Findings

#### Finding 1: CRITICAL — AC8 OpenAPI/Swagger (BEHOBEN)
- `springdoc-openapi-starter-webmvc-ui` 2.8.5 als Dependency in Parent-POM (dependencyManagement + Version-Property) und `odin-app/pom.xml` hinzugefuegt.
- Alle 3 Endpoints mit `@Operation`, `@ApiResponses`, `@ApiResponse`, `@Parameter`, `@Tag` annotiert.
- `/download`: 200/400/422/502 dokumentiert mit Schema-Referenzen auf Response-/Error-DTOs.
- `/cost-estimate`: 200/400/502 dokumentiert, Parameter mit Description und Example.
- `/ingest-log`: 200 dokumentiert mit ArraySchema, optionale Parameter mit Description.
- AC8 ist damit vollstaendig erfuellt.

#### Finding 2: MAJOR — DB-Tests fuer Repository-Methoden (BEHOBEN)
- 12 neue Zonky-Embedded-Postgres-Integrationstests in `DataIngestLogJdbcRepositoryIntegrationTest`:
  - `findAll`: 3 Tests (empty table, ordering, field mapping)
  - `findByDateRange`: 5 Tests (matching, no-match, boundary inclusive, cross-symbol, ordering)
  - `findBySymbol`: 2 Tests (matching, no-match)
  - `findBySymbolAndDateRange`: 2 Tests (matching, symbol-match-but-date-not)
- Gesamt jetzt 24 IT-Tests in der Klasse (12 bestehend + 12 neu), alle gruen.

#### Finding 3: MINOR — Unreachable default-Branch (BEHOBEN)
- `resolveBarIntervalSec()` jetzt exhaustiver switch ohne default: alle 6 DataSchema-Werte explizit gehandelt.
- OHLCV_1S/1M/1H/1D -> jeweiliger Intervall in Sekunden.
- MBP_1, TRADES -> throw IllegalArgumentException (defensive Absicherung).
- `DEFAULT_BAR_INTERVAL_SEC` Konstante entfernt (nicht mehr verwendet).
- Compiler erzwingt jetzt Anpassung bei neuen DataSchema-Werten.

### Remediation-Reviews (Runde 2)

#### ChatGPT-Review
- Empfiehlt: Error-Responses fuer /ingest-log ergaenzen (400/500), konsistente Schema-Werte (Enum vs API-Value), explizitere Datumssemantik in OpenAPI.
- Bewertet DB-Tests als solide. Empfiehlt zusaetzlich: deterministische Sortierung bei gleichem Tag, Null-Feld-Mapping in Findern, Range "start > end" Verhalten.
- Bewertung: Empfehlungen sind valide Verbesserungen, aber keine Blocker. Die 3 Findings sind sauber adressiert.

#### Gemini-Review Dimension 1 (Code-Bugs)
- LOW: hasDateRange ignoriert einzeln gesetzte Date-Parameter stillschweigend.
- Bewertung: Bekanntes Design — bewusste Entscheidung, da /ingest-log nur Query-Endpoint ist. Kann in separater Story verbessert werden.

#### Gemini-Review Dimension 2 (Konzepttreue)
- Finding 1 (AC8): **PASS**
- Finding 2 (DB-Tests): **PASS**
- Finding 3 (Switch): **PASS**

#### Gemini-Review Dimension 3 (Vollstaendigkeit)
- Keine neuen Regressionsrisiken identifiziert.
- springdoc 2.8.5 kompatibel mit Spring Boot 3.4.3 bestaetigt.
- Entfernung von DEFAULT_BAR_INTERVAL_SEC bricht keine bestehenden Tests (bestaetigt durch 73 gruene Tests).

### Test-Ergebnisse Runde 2
- Unit-Tests odin-app (DatabentoControllerTest): 27/27 PASSED
- Integration-Tests odin-app (DatabentoControllerIntegrationTest): 22/22 PASSED
- Integration-Tests odin-data (DataIngestLogJdbcRepositoryIntegrationTest): 24/24 PASSED
- Gesamt: 73 Tests, 0 Failures, 0 Errors
