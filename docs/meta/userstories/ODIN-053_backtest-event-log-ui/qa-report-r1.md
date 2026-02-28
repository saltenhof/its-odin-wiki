# QA Report — ODIN-053 (Runde 1)

**Story:** Backtest Event Log UI: Audit-Analyse & Laufübergreifende Auswertung
**Prüfdatum:** 2026-02-24
**Commits Backend:** d5b8a19 | **Commits Frontend:** f51eab0
**Bekannte Test-Erfolge:** Backend 296 Tests (0 Failures), Frontend 257 Tests (0 Failures)

---

## 1. Backend — Code-Qualität + Korrektheit (DoD 2.1)

### 1.1 EventQueryController.java

| Prüfpunkt | Ergebnis |
|-----------|----------|
| Kein `var` | PASS — explizite Typen durchgängig |
| Keine Magic Numbers | PASS — keine |
| JavaDoc auf allen public Methoden/Klassen | PASS — Klasse, Konstruktor, `queryEvents`, `buildBacktestContextCache`, Record-Methoden vollständig dokumentiert |
| Records für DTOs | PASS — `BacktestContextCache` als private record implementiert |
| Kein TODO/FIXME | PASS |
| `GET /api/v1/events` mit 5 optionalen Filtern | PASS — `instrumentId`, `eventType`, `from`, `to`, `emitter` via `@RequestParam(required = false)` |
| Pagination via `Pageable` | PASS — `@PageableDefault(size = 50)` |
| N+1-Problem vermieden | PASS — `BacktestContextCache` mit 2 gezielten Queries (kein N+1) |
| Native SQL Hibernate-6-Workaround | PASS — `Instant` → `String` → `CAST(:from AS timestamptz)` |

**Befund:** Der `q`-Freitext-Parameter (für `clientOrderId`/`brokerOrderId` Suche), der im Story-Scope für den Cross-Run-Endpoint genannt ist, fehlt in der Controller-Signatur und der Repository-Methode. Die Story beschreibt in den technischen Details: `GET /api/v1/events?...&q=clientOrderId`. Dieser Parameter ist nicht implementiert.

### 1.2 EventRecordDto.java

| Prüfpunkt | Ergebnis |
|-----------|----------|
| Record für DTO | PASS — `public record EventRecordDto(...)` |
| Neue Felder `backtestId` + `backtestName` | PASS — als nullable UUID/String |
| Factory-Methoden korrekt | PASS — `fromEntity(entity)` delegiert an `fromEntity(entity, null, null)` |
| JavaDoc | PASS |

**Befund:** Das Story-Interface (technische Details) spezifiziert `wallClockTimestamp: string` als Pflichtfeld des DTOs. Das Java-Backend-DTO und das TypeScript-Frontend-Interface enthalten dieses Feld nicht. Die `EventRecordEntity` hat das Feld (`getWallClockTimestamp()`), aber es wird nicht in das DTO gemappt und ist daher in der API-Response nicht verfügbar.

### 1.3 EventRecordRepository.java (odin-audit)

| Prüfpunkt | Ergebnis |
|-----------|----------|
| Neue `findByFilters`-Methode | PASS — vorhanden |
| Native SQL (Hibernate-6-Workaround) | PASS — `nativeQuery = true`, String-Parameter, `CAST(:from AS timestamptz)` |
| `countQuery` für Pagination | PASS — separates countQuery implementiert |
| `ORDER BY` fest in SQL (nicht via Pageable.sort) | PASS — `ORDER BY e.market_clock_timestamp ASC, e.sequence_number ASC` fest im SQL |
| JavaDoc | PASS |
| Kein TODO/FIXME | PASS |

### 1.4 BacktestRunRepository.java (odin-backtest)

| Prüfpunkt | Ergebnis |
|-----------|----------|
| Neue `findByBatchIdIn`-Methode | PASS — vorhanden |
| JPQL korrekt | PASS — `WHERE b.batchId IN :batchIds` |
| JavaDoc | PASS |

---

## 2. Backend — Tests (DoD 2.2 + 2.3)

| Anforderung | Ergebnis |
|-------------|----------|
| Unit-Tests `EventQueryControllerTest` | PASS — 5 Tests: empty result, backtest context enrichment, live run null context, filter forwarding, pagination |
| Integrationstests `EventQueryControllerIntegrationTest` (Zonky) | PASS — 7 Tests: Filter nach Instrument, Event-Typ, Zeitbereich, Emitter; Backtest-Context für Sim-Runs; null context für Live-Runs; Pagination |
| Test: Filter nach Instrument | PASS — `testFilterByInstrumentReturnsOnlyMatchingEvents` |
| Test: Filter nach Event-Typ | PASS — `testFilterByEventTypeReturnsOnlyMatchingEvents` |
| Test: Pagination | PASS — `testPaginationReturnsCorrectSlice` |

**Gesamt: 296 Backend-Tests, 0 Failures (BUILD SUCCESS)**

---

## 3. Frontend — Code-Qualität (DoD 2.1)

### 3.1 TypeScript strict (kein `any`)

| Datei | Ergebnis |
|-------|----------|
| EventLogTab.tsx | PASS — kein `any` |
| useEventLog.ts | PASS — nur Re-Export |
| EventTypeFilter.tsx | PASS — kein `any` |
| EventPayloadViewer.tsx | PASS — kein `any` |
| GlobalEventSearch.tsx | PASS — kein `any` |
| useGlobalEvents.ts | PASS — kein `any` (mock-Typen in Tests nutzen `unknown[]`) |
| backtesting.ts (neue Typen) | PASS — kein `any` |
| backtestingApi.ts (`getGlobalEvents`) | PASS |

### 3.2 CSS Modules (keine Inline-Styles)

| Datei | Ergebnis |
|-------|----------|
| EventLogTab.tsx | PASS — ausschließlich `styles.*` |
| EventPayloadViewer.tsx | PASS — ausschließlich `styles.*` |
| EventTypeFilter.tsx | PASS — ausschließlich `styles.*` |
| GlobalEventSearch.tsx | PASS — ausschließlich `styles.*` |

### 3.3 Feature→Feature-Import-Regel

| Prüfpunkt | Ergebnis |
|-----------|----------|
| `GlobalEventSearch.tsx` importiert `EventPayloadViewer` aus `../EventLogTab/` | PASS — beides liegt innerhalb von `src/domains/backtesting/`; kein Cross-Feature-Import |
| Keine Imports aus `dashboard`, `trading-operations` o.ä. | PASS |

### 3.4 Dateilocation

| Prüfpunkt | Ergebnis |
|-----------|----------|
| Alle neuen Dateien in `src/domains/backtesting/` | PASS |

### 3.5 Kein TODO/FIXME

PASS — keine TODO/FIXME-Kommentare in den neuen Dateien

---

## 4. Akzeptanzkriterien (AC-1 bis AC-9)

### AC-1: Tab "Events" in BacktestDetailPage

**PASS** — `BacktestDetailPage.tsx`:
- `DetailTab` Type enthält `'events'`
- Tab-Button "Events" vorhanden
- `{activeTab === 'events' && <EventLogTab backtestId={backtestId} />}` korrekt

### AC-2: Event-Typ-Dropdown dynamisch befüllt aus `/event-types`

**PASS** — `useBacktestEvents` ruft `api.getBacktestEventTypes(backtestId)` auf, die `GET /api/v1/backtests/{id}/event-types` anspricht. `EventTypeFilter` rendert die zurückgegebenen Typen dynamisch.

### AC-3: Freitextsuche vorhanden

**FAIL (PARTIAL)** — Die Story verlangt in der Backtest-Detail-Ansicht (EventLogTab) eine "Freitextsuche auf den Feldern `emitter`, `clientOrderId`, `brokerOrderId`". Die aktuelle Implementierung des EventLogTab bietet **nur** das Event-Typ-Dropdown (`EventTypeFilter`), aber kein Freitext-Suchfeld. In der Cross-Run-Ansicht (GlobalEventSearch) existieren zwar separate Felder für Emitter und Event-Typ, aber kein kombiniertes Freitext-Suchfeld für `clientOrderId`/`brokerOrderId`. AC-3 ist für den EventLogTab nicht erfüllt.

### AC-4: Payload-Viewer expandierbar

**PASS** — `EventPayloadViewer` rendert formatiertes JSON. Rows sind per `onClick`/`onKeyDown` togglebar. Default: eingeklappt (expand state initial `null`).

### AC-5: Chronologische Sortierung + umschaltbar

**PARTIAL PASS** — Standard-Sortierung ist chronologisch aufsteigend (Backend: `ORDER BY e.market_clock_timestamp ASC`). Jedoch ist die Sortierrichtung **nicht umschaltbar** — die Story verlangt ausdrücklich "Sortierrichtung umschaltbar". Es gibt keinen Sort-Toggle in EventLogTab. Im Backend ist die ORDER BY-Klausel fest eincodiert (nicht via Pageable.sort, da Hibernate-Bug). Dies ist ein bewusster Trade-off (protocol.md: "kein Sort via Pageable; kein Frontend-Sort nötig") — aber der AC verlangt "umschaltbar". **FAIL** für die Umschaltbarkeit.

### AC-6: Neuer `GET /api/v1/events` Endpoint vorhanden und filterbar

**PASS (PARTIAL)** — Endpoint ist vorhanden (`EventQueryController`), filterbar nach `instrumentId`, `eventType`, `from`, `to`, `emitter`. Der `q`-Parameter (Freitext für `clientOrderId`/`brokerOrderId`) aus der Story-Spezifikation fehlt, ist aber im AC-6 nicht explizit genannt (nur "Instrument, Event-Typ und Datumsbereich"). **PASS** für den AC-Wortlaut.

### AC-7: Cross-Run-Ansicht in UI vorhanden

**PASS** — `GlobalEventSearch` in `BacktestListPage` als zweiter Tab "Event Search" implementiert.

### AC-8: Link zum zugehörigen Backtest-Lauf in Cross-Run-Ansicht

**FAIL** — Die Story verlangt: "Jede Event-Zeile in der Cross-Run-Ansicht enthält einen Link zum zugehörigen Backtest-Lauf." In der aktuellen Implementierung (`GlobalEventSearch.tsx`, Zeile 175-181) wird der `backtestName` als statischer `<span>` angezeigt, mit `title={event.backtestId}` als Tooltip. Es ist **kein klickbarer Link** implementiert — weder ein `<a href>` noch `useNavigate` auf `/backtesting/{backtestId}`. Der `backtestId` ist im DTO vorhanden und könnte für eine Navigation genutzt werden, wird aber nicht verwendet.

### AC-9: data-testid Attribute

| Attribut | Gefunden | Datei |
|----------|----------|-------|
| `event-log-table` | PASS | `EventLogTab.tsx` Zeile 237: `data-testid="event-log-table"` |
| `event-type-filter` | PASS | `EventTypeFilter.tsx` Zeile 47: `data-testid="event-type-filter"` |
| `event-payload-viewer` | PASS | `EventPayloadViewer.tsx` Zeile 60: `data-testid="event-payload-viewer"` |
| `global-event-search` | PASS | `GlobalEventSearch.tsx` Zeile 265: `data-testid="global-event-search"` |
| `global-event-search-results` | PASS | `GlobalEventSearch.tsx` Zeile 372: `data-testid="global-event-search-results"` |

**AC-9: PASS** — Alle 5 geforderten `data-testid`-Attribute vorhanden.

---

## 5. Tests (DoD 2.2) — Frontend Custom Hooks

| Anforderung | Ergebnis |
|-------------|----------|
| Unit-Tests für `useGlobalEvents` | PASS — 9 Tests in `useGlobalEvents.test.ts` |
| Unit-Tests für `useEventLog` / `useBacktestEvents` | **FAIL** — Kein Test für `useBacktestEvents.ts` vorhanden. `useEventLog.ts` ist nur ein Re-Export. Es gibt keine `useBacktestEvents.test.ts` oder `useEventLog.test.ts` im Projekt. Die DoD 2.2 fordert "Unit-Tests für neue Frontend-Custom-Hooks (gemockte API-Calls)". |

---

## 6. Protokolldatei (DoD 2.7)

Die Protokolldatei `protocol.md` enthält:
- Story-Kurzfassung mit allen Parts (A, B, C, D) ✓
- Backend-Abschnitt mit geänderten Dateien und Test-Ergebnissen ✓
- Frontend-Abschnitt mit neuen Dateien ✓
- ChatGPT Review (DoD 2.5) mit Findings und Maßnahmen ✓
- Bekannte Trade-offs / Technische Schulden ✓
- Fehler und Korrekturen während Implementierung ✓

**FAIL (DoD 2.6):** Das protocol.md enthält **keine Notiz** darüber, dass der Gemini-Review dauerhaft entfällt. Laut QA-Auftrag gilt DoD 2.6 als erfüllt nur wenn "protocol.md eine Notiz darüber enthält". Diese Notiz fehlt.

---

## 7. Zusammenfassung der Findings

| # | Schwere | Befund | AC / DoD |
|---|---------|--------|----------|
| F-1 | **MEDIUM** | AC-3 (EventLogTab): Freitextsuche auf `emitter`, `clientOrderId`, `brokerOrderId` fehlt im EventLogTab. Nur Event-Typ-Dropdown vorhanden. | AC-3 |
| F-2 | **MEDIUM** | AC-5: Sortierrichtung nicht umschaltbar. ORDER BY ist fest im Backend-SQL. Kein Sort-Toggle in der UI. | AC-5 |
| F-3 | **MEDIUM** | AC-8: Kein Link zum Backtest-Lauf in der Cross-Run-Ansicht. `backtestName` ist statischer Text, kein `<a>`/`useNavigate`. | AC-8 |
| F-4 | **LOW** | DoD 2.2: `useBacktestEvents` hat keinen Unit-Test. Nur `useGlobalEvents` ist getestet. | DoD 2.2 |
| F-5 | **LOW** | DoD 2.6: `protocol.md` fehlt Notiz über dauerhaften Wegfall des Gemini-Reviews. | DoD 2.6 |
| F-6 | **LOW** | Story-Spec-Abweichung: `wallClockTimestamp` im Story-Interface spezifiziert, aber nicht in Java-DTO (`EventRecordDto.java`) und TypeScript-Typ gemappt. Feld existiert in der Entity, wird aber nicht serialisiert. | DoD 2.1 |
| F-7 | **LOW** | Story-Spec-Abweichung: `q`-Parameter (Freitext) in der Backend-Endpoint-Spezifikation genannt, aber weder im Controller noch im Repository implementiert. Durch individuelle Filter-Parameter teilweise abgedeckt, aber nicht vollständig (keine `clientOrderId`/`brokerOrderId`-Suche). | Technische Details |

---

## 8. Bewertung je DoD

| DoD | Status |
|-----|--------|
| 2.1 Code-Qualität (Backend) | PASS (mit Anmerkungen F-6, F-7) |
| 2.1 Code-Qualität (Frontend) | PASS |
| 2.2 Unit-Tests | FAIL (F-4) |
| 2.3 Integrationstests | PASS |
| 2.5 ChatGPT-Sparring | PASS |
| 2.6 Gemini-Notiz in protocol.md | FAIL (F-5) |
| 2.7 Protokolldatei | PASS (Inhalt vollständig) |
| AC-1 | PASS |
| AC-2 | PASS |
| AC-3 | FAIL (F-1) |
| AC-4 | PASS |
| AC-5 | FAIL (F-2) |
| AC-6 | PASS |
| AC-7 | PASS |
| AC-8 | FAIL (F-3) |
| AC-9 | PASS |

---

QS-SIGNAL: FAIL
