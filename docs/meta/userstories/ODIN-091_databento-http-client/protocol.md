# Protokoll: ODIN-091 — Databento HTTP Client (Retrofit2 + OkHttp3)

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (33 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (8 Tests)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### Package-Struktur: `de.its.odin.data.databento`
Databento-spezifischer Code als Sub-Domain unter odin-data, mit Sub-Packages:
- `client/` — DabentoApi (Retrofit interface), DabentoApiFactory
- `config/` — DabentoProperties (@ConfigurationProperties Record)
- `interceptor/` — DabentoAuthInterceptor, DabentoRetryInterceptor
- `exception/` — DabentoApiException hierarchy, DatabentoCostLimitException

Begruendung: Klare Trennung der Databento-Integrationsschicht von der bestehenden Data-Pipeline.
Die Sub-Domain-Packages sind erlaubt laut module-structure.md Abschnitt 2.2.

### Interceptor-Reihenfolge
1. DabentoRetryInterceptor (Application-Interceptor) — Wraps den gesamten Request-Zyklus
2. DabentoAuthInterceptor (Network-Interceptor) — Wird pro Network-Request angewendet
3. HttpLoggingInterceptor (Application-Interceptor) — Loggt Headers, redacted Authorization

Begruendung: Auth als Network-Interceptor, damit er auch bei Redirects greift.
Retry als Application-Interceptor, damit er den gesamten Request inklusive Auth wiederholt.

### LockSupport statt Thread.sleep
DabentoRetryInterceptor verwendet LockSupport.parkNanos() statt Thread.sleep(),
wie in den Story-Notizen empfohlen. Vermeidet InterruptedException-Handling.
Thread.interrupted() Check nach parkNanos hinzugefuegt (Gemini Dim1 Finding).

### Authorization Header Redaction
HttpLoggingInterceptor redacted den Authorization-Header via .redactHeader("Authorization").
API-Key erscheint nicht in Logs.

### Keine @Component-Annotation auf DabentoApiFactory
odin-data ist ein Pro-Pipeline-Modul, keine Spring-Annotations (keine @Component, @Service).
DabentoApiFactory wird in odin-app per @Bean instanziiert (nicht Teil dieser Story).

### converter-jackson beibehalten
Obwohl die meisten Responses als ResponseBody (Raw) konsumiert werden, ist converter-jackson
fuer potenzielle zukuenftige JSON-Response-Typen nuetzlich (z.B. getCost Structured Response).

### Exception-Hierarchie: DatabentoCostLimitException extends RuntimeException
DatabentoCostLimitException ist bewusst kein DabentoApiException-Subtyp.
Es handelt sich um einen clientseitigen Limit-Check, nicht um einen HTTP-Fehler.
Gemini Dimension 2 hat dies als logisch korrekte Abweichung bestaetigt.

## Offene Punkte
- Retry-After Header: DabentoRetryInterceptor ignoriert den Retry-After Header bei 429.
  Aktuell exponentieller Backoff. Koennte in Zukunft ergaenzt werden (ChatGPT + Gemini Dim3).
- Retry bei Streaming: Retry-Interceptor retried den gesamten Request. Fuer grosse
  Streaming-Downloads soll die Recovery im DatabentoBatchService (separate Story) stattfinden,
  nicht auf HTTP-Client-Ebene.

## ChatGPT-Sparring

### Vorgeschlagene Szenarien (DoD 5.5)
1. **Retry backoff overflow** — Acknowledged, maxRetries ist @Min(0), realistisch max 3-5
2. **Mixed retriable -> non-retriable** — UMGESETZT: Test retriableThenNonRetriableStopsImmediately()
3. **Status code boundary 499** — UMGESETZT: Test status499IsNotRetried()
4. **Status 504 retried** — UMGESETZT: Test status504IsRetried()
5. **costLimitUsd @NotNull missing** — UMGESETZT: @NotNull hinzugefuegt
6. **Blank API key** — UMGESETZT: IllegalArgumentException in DabentoAuthInterceptor + Tests
7. **Base URL trailing slash** — UMGESETZT: Test createThrowsWhenBaseUrlMissingTrailingSlash()
8. **Redirect credential leak** — Acknowledged, Databento API redirected nicht zu fremden Hosts
9. **Retry-After header** — Deferred to future enhancement
10. **I/O exception retry** — Deliberately excluded (retryOnConnectionFailure=false by design)

Ergebnis: 6 von 10 Szenarien umgesetzt, 4 begruendet verworfen/deferred.

## Gemini-Review

### Dimension 1: Code-Review
- **HIGH: Thread Interruption Bug** — BEHOBEN: Thread.interrupted() check nach parkNanos()
- **HIGH: Exception hierarchy not thrown** — By design: Exceptions werden in der Service-Schicht
  (DatabentoBatchService, separate Story) geworfen, nicht im HTTP-Client selbst
- **MEDIUM: Read timeout @Min(1)** — Default 300s ist korrekt, @Min(1) ist Guard gegen 0/negative
- **LOW: Auth interceptor** — Als korrekt bestaetigt

### Dimension 2: Konzepttreue
- Alle 11 Akzeptanzkriterien: PASS
- Minor: DatabentoCostLimitException ohne HTTP-Status — logisch korrekt, kein HTTP-Fehler
- Minor: Defaults in odin-data.properties statt application.properties — CSpec-konform

### Dimension 3: Praxis
- **Retry from Scratch Cost Trap** — Recovery in DatabentoBatchService, nicht HTTP-Client
- **Retry-After Header** — Deferred, exponentieller Backoff genuegt initial
- **Read Timeout vs Server Queuing** — 300s Default ist ausreichend
- **Connection Leaks** — BEHOBEN: getRange() JavaDoc warnt vor Resource-Leak
- **Error Body Parsing** — getRange() JavaDoc weist auf JSON-Error-Body hin
