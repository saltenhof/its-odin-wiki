# ODIN-039: Daily Report und Trade Journal Endpoints

**Modul:** odin-app
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** ODIN-029 (Daily Performance Recording)
**Geschaetzter Umfang:** S

---

## Kontext

Das Konzept definiert Report-Artefakte: Daily Report, Trade Journal, Long-Term-Metriken. Der REST-API fehlen Endpoints zum Abruf dieser Daten. Das Frontend braucht sie fuer History-Ansichten und Performance-Tracking. Diese Story implementiert die Backend-API-Schicht — die Daten selbst werden durch ODIN-029 persistiert.

## Scope

**In Scope:**
- `GET /api/v1/reports/daily/{date}` — Tagesbericht
- `GET /api/v1/reports/trades/{date}` — Trade-Journal fuer einen Tag
- `GET /api/v1/reports/performance?from={date}&to={date}` — Performance-Uebersicht ueber Zeitraum
- DTOs fuer Daily Report, Trade Journal Entry, Performance Summary

**Out of Scope:**
- PDF-Export (JSON in V1)
- Asynchrone Report-Generierung (synchroner REST-Call in V1)
- Email-Versand von Reports
- Report-Caching

## Akzeptanzkriterien

- [ ] Daily Report DTO: `totalPnl`, `winRate`, `tradeCount`, `maxDrawdown`, `sharpe`, `cycleCount`, `alerts`
- [ ] Trade Journal DTO: Liste von `TradeJournalEntryDto` mit `entry/exit/pnl/reason/duration/rMultiple`
- [ ] Performance Summary DTO: Kumulierte Metriken ueber angefragten Zeitraum
- [ ] HTTP 404 wenn kein Report fuer das angefragte Datum existiert
- [ ] Pagination fuer Trade-Liste (wenn > 100 Trades: Page-Parameter)

## Technische Details

**Dateien:**
- `odin-app/src/main/java/de/its/odin/app/controller/ReportController.java` (neue Klasse)
- `odin-app/src/main/java/de/its/odin/app/dto/DailyReportDto.java` (neues Record)
- `odin-app/src/main/java/de/its/odin/app/dto/TradeJournalEntryDto.java` (neues Record)
- `odin-app/src/main/java/de/its/odin/app/dto/PerformanceSummaryDto.java` (neues Record)

**Datenquellen:** `DailyPerformanceRepository` und `TradeRepository` (aus odin-core/odin-execution).

## Konzept-Referenzen

- `docs/concept/10-observability.md` — Abschnitt 7 "Report-Artefakte"
- `docs/concept/10-observability.md` — Abschnitt 7.1 "Trade Journal"
- `docs/concept/10-observability.md` — Abschnitt 7.2 "Daily Report"

## Guardrail-Referenzen

- `docs/frontend/guardrails/frontend.md` — "REST Client"
- `T:\codebase\its_odin\CLAUDE.md` — Records fuer DTOs, explizite Typen
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-app`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `DEFAULT_PAGE_SIZE`, `MAX_TRADES_PER_PAGE`)
- [ ] Records fuer DTOs (`DailyReportDto`, `TradeJournalEntryDto`, `PerformanceSummaryDto`)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.app.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `ReportController`: Daily-Report-Endpoint mit Mock-Repository — korrektes DTO, 404 bei fehlendem Datum
- [ ] Unit-Tests fuer Trade-Journal-Endpoint: Pagination korrekt (Seite 1, Seite 2, leere Seite)
- [ ] Unit-Tests fuer Performance-Summary-Endpoint: Zeitraum-Aggregation korrekt
- [ ] Unit-Tests fuer R-Multiple-Berechnung: `(exitPrice - entryPrice) / (entryPrice - initialStop)`
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Repositories
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: HTTP GET auf `/api/v1/reports/daily/{date}` mit echtem Repository-Stub (echte Daten, kein Mock) — korrektes JSON-Response
- [ ] Integrationstest: 404-Handling bei nicht vorhandenem Datum
- [ ] Integrationstest: Trade-Journal mit Pagination — korrektes Paging-Verhalten mit realen Daten
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass ReportController, Repositories und DTO-Mapping korrekt zusammenarbeiten.

### 2.4 Tests — Datenbank

- [ ] Embedded-Postgres-Tests mit **Zonky** (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migrationen werden im Test ausgefuehrt
- [ ] `DailyPerformanceRepository` und `TradeRepository` werden gegen echte DB getestet (nicht nur Mocks)
- [ ] Testdaten: Mindestens ein vollstaendiger Tagesbericht in DB, Abfrage prueft Korrektheit der Werte
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

**Referenz:** CLAUDE.md → Tests (`*IntegrationTest` fuer Failsafe)

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `ReportController.java`, allen DTO-Klassen, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei einem Tag ohne Trades? Was wenn `initialStop` null ist (R-Multiple undefiniert)? Was bei einem sehr langen Zeitraum in der Performance-Summary (Performance)?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Division-by-Zero bei R-Multiple-Berechnung, korrektes 404-Handling, Null-Safety bei optionalen DTO-Feldern, N+1-Query-Probleme bei Trade-Journal"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/10-observability.md` Abschnitte 7, 7.1, 7.2 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle DTO-Felder dem Konzept entsprechen. Sind Daily Report und Trade Journal vollstaendig? Stimmt das Pagination-Verhalten mit Frontend-Erwartungen ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Welche Report-Anforderungen entstehen in der Praxis, die hier nicht behandelt sind? Z.B. Timezone-Handling (ETZ vs. lokale Zeit), Benchmarking gegen Index, Steuerreporting-relevante Felder?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Pagination-Strategie, R-Multiple bei fehlendem Stop)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- Daten kommen aus `DailyPerformanceRepository` (ODIN-029 muss abgeschlossen sein) und `TradeRepository`
- R-Multiple pro Trade: `(exitPrice - entryPrice) / (entryPrice - initialStop)` — Division-by-Zero abfangen wenn `initialStop == entryPrice`
- Timezone: Alle Datumsfelder als `LocalDate` (Handelstag, keine Uhrzeit) verwenden
