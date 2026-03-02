# Protokoll: ODIN-090 — HistoricalDataProvider Port Interface

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (61 Tests, alle gruen)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases (13 Tests hinzugefuegt)
- [x] Integrationstests geschrieben (8 Tests, alle gruen)
- [x] Gemini-Review Dimension 1 (Code) — keine Aenderungen noetig
- [x] Gemini-Review Dimension 2 (Konzepttreue) — alle Abweichungen by-design
- [x] Gemini-Review Dimension 3 (Praxis) — Findings bewertet, alle by-design oder out-of-scope
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Deliverables

| Datei | Typ | Package |
|-------|-----|---------|
| `HistoricalDataProvider.java` | Port-Interface | `de.its.odin.api.port` |
| `HistoricalDataRequest.java` | DTO Record | `de.its.odin.api.dto` |
| `DataIngestResult.java` | DTO Record | `de.its.odin.api.dto` |
| `DataSchema.java` | Enum | `de.its.odin.api.model` |
| `DataProvider.java` | Enum | `de.its.odin.api.model` |
| `DataSchemaTest.java` | Unit-Test (15 Tests) | `de.its.odin.api.model` |
| `DataProviderTest.java` | Unit-Test (3 Tests) | `de.its.odin.api.model` |
| `HistoricalDataRequestTest.java` | Unit-Test (27 Tests) | `de.its.odin.api.dto` |
| `DataIngestResultTest.java` | Unit-Test (16 Tests) | `de.its.odin.api.dto` |
| `HistoricalDataProviderIntegrationTest.java` | Integrationstest (8 Tests) | `de.its.odin.api.port` |

**Gesamt:** 69 Tests (61 Unit + 8 Integration), alle gruen. Full-Project-Build erfolgreich (10 Module).

## Design-Entscheidungen

### 1. DTO-Platzierung: `de.its.odin.api.dto` statt `de.its.odin.api.port`
Die Issue-Beschreibung schlaegt `de.its.odin.api.port` oder `de.its.odin.api.model` vor.
Entscheidung: `HistoricalDataRequest` und `DataIngestResult` liegen in `de.its.odin.api.dto`,
weil sie DTOs sind und das dem existierenden Muster folgt (OrderRequest, TradeIntent etc. liegen
ebenfalls in `dto/`). Die Enums (DataSchema, DataProvider) liegen korrekt in `de.its.odin.api.model`.

### 2. Immutabilitaet: Defensive Copy in HistoricalDataRequest
`HistoricalDataRequest.symbols` wird im Compact Constructor via `List.copyOf()` defensiv kopiert.
Dies verhindert nachtraegliche Mutation durch den Aufrufer und ist konsistent mit dem Muster
in anderen odin-api Records (z.B. IndicatorResult.flags).

### 3. Null-Safety: requireNonNull im Compact Constructor
Beide Records (HistoricalDataRequest, DataIngestResult) validieren alle Referenz-Felder im
Compact Constructor mit Objects.requireNonNull(). Fail-fast statt spaete NullPointerExceptions.

### 4. DataIngestResult.schema: DataSchema Enum statt String
Die Konzeptvorlage verwendet `String schema`. Aenderung zu `DataSchema schema`, weil die Story
explizit fordert "ENUM statt String fuer endliche Mengen" und der DataSchema-Enum genau dafuer
existiert. Typ-sicher und konsistent.

### 5. Benennungskonvention: Kein DabentoRequest
Das Konzeptdokument nennt den Request `DabentoRequest`. Die Story benennt ihn korrekt in
`HistoricalDataRequest` um, da das Port-Interface provider-neutral sein soll.

### 6. Methoden-Signaturen: estimateCost(DTO) vs. download*(flat params)
Bewusste Design-Entscheidung aus der Issue-Spezifikation: estimateCost akzeptiert das flexible
HistoricalDataRequest-DTO (Multi-Symbol, beliebige Schema-Kombinationen), waehrend die
Download-Methoden flat parameters verwenden fuer Klarheit und tagesweise Idempotenz.

### 7. LocalDate fuer Zeitraeume
LocalDate statt Instant/ZonedDateTime, weil Handelstage Kalendertage sind. Die Zeitzonen-
Konvertierung (Eastern Time, UTC) wird in der Implementierung (odin-data) behandelt, nicht
auf der Vertragsebene. Konsistent mit der Konzeptvorlage.

## Offene Punkte
Keine.

## ChatGPT-Sparring

**Durchgefuehrt:** 2026-03-01. Alle Production- und Test-Dateien an ChatGPT uebermittelt.
Prompt-Fokus: API-Vollstaendigkeit, Factory-Method-Edge-Cases, BigDecimal-Praezision, Test-Coverage-Luecken.

### ChatGPT-Findings im Original

**1. API completeness (MBP_1, TRADES, OHLCV variants)**
- Interface ist intern konsistent, aber asymmetrisch: `estimateCost()` akzeptiert das volle `HistoricalDataRequest`-DTO (Multi-Symbol, Dataset, Encoding), waehrend die Download-Methoden nur flat params nehmen. Multi-Symbol-Downloads sind damit nicht ueber das Interface moeglich.
- `downloadBars(..., int barIntervalSec)` impliziert beliebige Sekunden, `DataSchema` unterstuetzt aber nur diskrete Werte (1/60/3600/86400). Nicht-unterstuetzte Werte werden im Stub still auf `OHLCV_1M` gemapped.

**2. Edge cases in factory methods**
- `ohlcv1m(symbol, start, end)` mit `start.isAfter(end)` ist aktuell erlaubt (kein Guard).
- Leere `symbols`-Liste via Konstruktor erlaubt (Provider lehnt dies moeglicherweise ab).
- Blank-String fuer `symbol` / `dataset` / `encoding` / `stypeIn` wird nicht abgefangen.

**3. BigDecimal precision**
- JavaDoc spezifiziert 4dp, aber `DataIngestResult` erzwingt den Scale nicht im Compact Constructor.
- `BigDecimal.equals()` ist scale-sensitiv — `1.0 != 1.0000`. Producer muss konsistenten Scale liefern.
- Empfehlung (staerkste bis schwaechste): (1) `long costMicros`, (2) Scale im Konstruktor canonicalisieren, (3) `scale() == 4` validieren.

**4. Test coverage gaps (zusaetzliche Szenarien)**
- `downloadBars` mit ungueltigen Intervallen (0, -1, unstuetzte Werte) — aktuell nicht getestet.
- Blank-String-Validierung fuer `symbol`, `dataset`, `encoding`, `stypeIn`.
- `rowCount < 0`, `costUsd < 0`, `downloadDuration.isNegative()` ablehnen.
- `DataProvider` enum fehlt in `DataIngestResult`.

**5. Design issues**
- `DataProvider` enum in `DataIngestResult` nicht enthalten — gut als Audit-Identifier, fehlt im Result.
- `barIntervalSec: int` statt `Duration` — `Duration` waere selbstdokumentierender.

**6. Java 21 compliance**
- Records + defensive copy sind idiomatisch und sauber.
- `getFirst()` / `getLast()` in Tests korrekt fuer Java 21.

### Bewertung und Entscheidungen (ChatGPT)

**Verworfen — BY DESIGN:**
- Asymmetrische API (DTO vs. flat params): Bewusste Design-Entscheidung aus der Issue-Spezifikation (#6 oben). Tagesweise Idempotenz und Fehler-Isolation sind wichtiger als API-Symmetrie.
- `barIntervalSec: int`: Issue-Spezifikation definiert explizit `int barIntervalSec`. `Duration` waere eleganter, ist aber Refactoring-Aufwand ohne funktionalen Mehrwert fuer diese Story.
- Blank-String-Validierung: DTOs in `odin-api` ueber-validieren bewusst nicht. Provider-spezifische Validierung gehoert in die Implementierung (odin-data).
- `startDate > endDate` Validierung: DTO ist Datentraeger, keine Business-Rule-Enforcement.
- `DataProvider` in `DataIngestResult`: Out of Scope dieser Story. DataProvider-Tracking kann bei der Persistenz-Story in odin-data ergaenzt werden.

**Verworfen — Out of Scope:**
- BigDecimal-Scale-Canonicalisierung: Over-Engineering fuer ein DTO. Producer setzt korrekten Scale. Verhalten bereits im Test `bigDecimalScale_affectsEquality_documentedBehavior` dokumentiert.
- Negative rowCount/costUsd Validierung: Record ist Datentraeger, keine Business-Rule-Enforcement.

**Keine Code-Aenderungen erforderlich.** Alle von ChatGPT identifizierten Test-Szenarien waren bereits in den 13 zusaetzlichen Tests umgesetzt, die in der Originalimplementierung hinzugefuegt wurden.

---

## Gemini-Review

**Durchgefuehrt:** 2026-03-01. Drei separate Gemini-Sessions (je eine pro Dimension).

### Dimension 1: Code-Review

**Gemini-Findings im Original:**

- **CRITICAL — Inkonsistente API-Design:** `estimateCost()` nutzt `HistoricalDataRequest`-DTO, Download-Methoden verwenden einzelne Parameter. Dies "hebelt den Zweck des umfassenden Request-Objekts aus."
- **MAJOR — Separation of Concerns:** Interface-Dokumentation fordert Download + Persist als eine atomare Operation. Eine "Provider"-Schnittstelle sollte idealerweise nur fuer Datenabruf zustaendig sein, nicht fuer Persistenz.
- **MAJOR — Databento-spezifische Defaults in DTO:** `HistoricalDataRequest` haelt Konstanten wie `XNAS.ITCH` und `raw_symbol` — nicht provider-neutral.
- **MAJOR — DataSchema Abstraktionsleck:** `toApiValue()` liefert den exakten Databento-API-String. Das koppelt odin-api an einen spezifischen Provider.
- **MINOR — Dataset fehlt in Downloads:** `HistoricalDataRequest` enthaelt `dataset`-Feld, Download-Methoden ignorieren es.
- **MINOR — DataProvider fehlt in DataIngestResult.**
- **INFO — Starke Null-Safety und Immutabilitaet:** Positiv bewertet.
- **INFO — Factory Methods:** Konvenient und klar.

**Bewertung (Dimension 1):**

- CRITICAL asymmetrische API: BY DESIGN per Issue-Spezifikation. Download-Methoden mit flat params sind explizit so spezifiziert — Tagesweise Idempotenz und Fehler-Isolation.
- MAJOR Separation of Concerns: BY DESIGN. Das Port-Interface fuer einen "Historical Data Provider" ist per Definition zustaendig fuer den gesamten Ingest-Workflow (Download + Persist). Der Name drueckt dies aus. Einen separaten "Persister" zu extrahieren waere eine Abstraktionsschicht ohne Mandat (CLAUDE.md: "no abstraction layers without mandate").
- MAJOR Databento-Defaults: BY DESIGN. Die Defaults in `HistoricalDataRequest` sind konfigurierbar (kein `final` im Record-Sinn — der Aufrufer kann sie ueberschreiben). Sie erleichtern die haeufigste Nutzung (XNAS.ITCH). Vollstaendige Provider-Neutralitaet auf DTO-Ebene wuerde bedeuten, dass keine sinnvollen Defaults existieren.
- MAJOR DataSchema Abstraktionsleck: BY DESIGN. `DataSchema` modelliert die Databento-Schemas — das ist kein Leck, das ist der Scope dieser Story. Die Story heisst explizit "Databento-Integration". `toApiValue()` ist bewusst Databento-spezifisch.
- MINOR DataProvider fehlt in DataIngestResult: INFORMATIONAL, Out of Scope.

**Keine Code-Aenderungen erforderlich.**

### Dimension 2: Konzepttreue

**Gemini-Findings im Original:**

- **CRITICAL — DataSchema als typed Enum statt String:** Konzept definiert `schema` als `String` in `DabentoRequest`. Implementierung einfuehrt typsicheres Enum mit zusaetzlichen OHLCV-Intervallen (1S, 1H, 1D) die im Konzept nicht explizit aufgelistet sind.
- **MAJOR — Download-Methoden ignorieren Schema:** Download-Methoden nehmen kein `DataSchema`-Argument, obwohl das DTO es enthaelt (nur fuer `estimateCost`). Erzeugt Diskrepanz.
- **MINOR — DataProvider fehlt in DataIngestResult:** Konzeptvorlage verwendet `String schema` — Implementierung hat `DataSchema schema` (positiv). Aber `DataProvider`-Referenz fehlt im Result.
- **INFO — Intentional Renaming DabentoRequest → HistoricalDataRequest:** Positiv bewertet.
- **INFO — Factory Methods:** Erfolgreich implementiert.

**Bewertung (Dimension 2):**

- CRITICAL DataSchema als Enum statt String: BY DESIGN. CLAUDE.md-Guardrail: "ENUM statt String fuer endliche Mengen." Die Enum-Varianten (1S, 1H, 1D) sind im Issue-Scope explizit genannt (`DataSchema (MBP_1, TRADES, OHLCV_1S, OHLCV_1M, OHLCV_1H, OHLCV_1D)`) — nicht dem Konzeptdokument, sondern der Story-Spezifikation folgend. Das Konzeptdokument ist eine Vorlage, die Story-Spezifikation ist bindend.
- MAJOR Download-Methoden ignorieren Schema: BY DESIGN. Jede Download-Methode ist auf genau ein Schema festgelegt (`downloadQuotes` → MBP_1, `downloadTrades` → TRADES, `downloadBars` → OHLCV). Das ist klarer als ein generisches `download(request)`.
- MINOR DataProvider: Out of Scope.

**Keine Code-Aenderungen erforderlich.**

### Dimension 3: Praxis-Review

**Gemini-Findings im Original:**

- **CRITICAL — Fehlendes Dataset-Parameter in Download-Methoden:** `HistoricalDataRequest` enthaelt `dataset`, Download-Methoden ignorieren es. Kein Weg fuer den Aufrufer, das Exchange/Dataset beim eigentlichen Download anzugeben.
- **CRITICAL — Partial Downloads & Transaction State:** Databento kann HTTP 206 Partial Content zurueckliefern, oder ein massiver Streaming-Download kann auf halbem Weg fehlschlagen. Interface-Vertrag definiert nicht, ob partielle Zustaende atomar zurueckgerollt werden oder ob partielle Daten in der DB verbleiben.
- **MAJOR — Unklarer Error-Handling-Vertrag:** Interface definiert keine checked Exceptions. Databento liefert verschiedene HTTP-Codes (400, 403, 429, 504). Aufrufer kann transiente Netzwerkfehler nicht von fatalen Auth-/Parameter-Fehlern unterscheiden.
- **MAJOR — Cost Estimate vs. Actual Billing:** Databento berechnet nach tatsaechlichem Verbrauch/gelieferten Bytes, was leicht von Schaetzungen abweichen kann. Bei strikten Kostengrenzen koennte dies Annahmen brechen.
- **MINOR — Zero-Row bei Wochenenden/Feiertagen:** Interface spezifiziert nicht explizit, ob ein leerer Provider-Response (Wochenende, Feiertag) ein erfolgreiches `DataIngestResult` mit `rowCount=0` oder eine Exception ist.
- **INFO — Thread-Blocking bei riesigen Datasets:** Synchroner MBP-1 Tick-Download kann den aufrufenden Thread 30+ Minuten blockieren. Dokumentiert im Interface (explizit "synchronous and blocking"), aber Aufrufer muessen dies beachten.

**Bewertung (Dimension 3):**

- CRITICAL Dataset fehlt in Downloads: BY DESIGN. Download-Methoden verwenden den Default-Dataset (XNAS.ITCH), der in der Implementierung (odin-data) aus der Konfiguration (`odin.data.databento.default-dataset`) gelesen wird. Das Interface-Design priorisiert Einfachheit fuer den haeufigsten Use Case (Single-Dataset, Single-Symbol). Wer ein anderes Dataset benoetigt, kann das Interface erweitern — YAGNI fuer diese Story.
- CRITICAL Partial Downloads/Transaction State: ACKNOWLEDGED als kuenftiges Implementierungsthema, aber bewusst Out of Scope fuer das Port-Interface. Das Interface definiert den Vertrag auf Ebene "erfolgreich" vs. "Exception". Transaktionsverhalten ist Implementierungs-Entscheidung in odin-data (z.B. pro-Tag-Transaktion mit Rollback). Das Interface-Javadoc sagt "persists it to the database" — das impliziert Atomizitaet pro Methodenaufruf. Wenn die Implementierung das nicht garantiert, muss sie eine Exception werfen.
- MAJOR Error-Handling-Vertrag: ACKNOWLEDGED. Das Interface verwendet bewusst unchecked Exceptions (kein throws-Clause). Implementierungen in odin-data werden provider-spezifische Runtime-Exceptions werfen (DatabentApiException etc.). Das Port-Interface in odin-api haelt sich heraus — es ist provider-neutral. Dokumentation im JavaDoc ("synchronous and blocking") gibt ausreichend Hinweise.
- MAJOR Cost Estimate Accuracy: INFORMATIONAL. Databento-Kostenschaetzungen sind typischerweise genau fuer unser Volumen. Geringfuegige Abweichungen (Cents) sind akzeptabel. Das Interface-Design (`BigDecimal costUsd`) laesst Raum fuer "actual or estimated" — die `DataIngestResult`-JavaDoc spezifiziert das korrekt.
- MINOR Zero-Row Weekends/Holidays: ACKNOWLEDGED als Implementierungsdetail. Die Implementierung in odin-data wird `rowCount=0` zurueckgeben (kein Fehler). Das ist impliziert durch den Vertrag: erfolgreich = keine Exception.
- INFO Thread-Blocking: Bereits im Interface-JavaDoc dokumentiert ("synchronous and blocking").

**Keine Code-Aenderungen erforderlich.** Alle Praxis-Findings sind entweder by-design, explizit out-of-scope fuer diese Story, oder Implementierungsdetails die in odin-data behandelt werden.

### Gesamtbewertung Gemini-Review
Alle Findings wurden bewertet. Keine Code-Aenderungen erforderlich. Die Implementierung ist konsistent mit der Issue-Spezifikation, den CLAUDE.md-Guardrails und dem Konzeptdokument (mit dokumentierten, begruendeten Abweichungen).
