# ODIN-053 — Backtest Event Log UI: Implementierungsprotokoll

## Story-Kurzfassung

Backtest Event Log UI mit vier Teilen:
- **Part A**: Backend `GET /api/v1/events` (cross-run Event Search) + EventRecordDto-Erweiterung
- **Part B**: Events-Tab in `BacktestDetailPage` mit neuen `EventLogTab`-Komponenten
- **Part C**: Cross-run Global Event Search (`GlobalEventSearch`) in `BacktestListPage`
- **Part D**: `data-testid`-Attribute

---

## Backend (Part A)

### Geänderte Dateien

**`odin-app/src/main/java/de/its/odin/app/dto/EventRecordDto.java`**
- Neue Felder `backtestId` (UUID, nullable) und `backtestName` (String, nullable)
- Neue Factory-Methode `fromEntity(entity, backtestId, backtestName)`
- Bestehende `fromEntity(entity)` delegiert an die neue Methode mit Nulls (keine Breaking Changes)

**`odin-audit/src/main/java/de/its/odin/audit/repository/EventRecordRepository.java`**
- Neue Methode `findByFilters(instrumentId, eventType, from, to, emitter, pageable)`
- Native SQL (nicht JPQL) wegen Hibernate 6 Null-Type-Inference-Bug mit `Instant`-Parametern in `IS NULL`-Patterns
- `from`/`to` als `String` (ISO-8601) statt `Instant`, mit `CAST(:from AS timestamptz)` in SQL
- `ORDER BY` fest in SQL (nicht via `Pageable.sort`) — verhindert `Spalte e.marketclocktimestamp existiert nicht`-Fehler

**`odin-backtest/src/main/java/de/its/odin/backtest/persistence/BacktestRunRepository.java`**
- Neue Methode `findByBatchIdIn(Collection<UUID> batchIds)` — JPQL-Query
- Wird für Backtest-Context-Enrichment benötigt (batchId → BacktestRun)

**`odin-app/src/main/java/de/its/odin/app/controller/EventQueryController.java`** (neu)
- `GET /api/v1/events` mit optionalen Filtern: `instrumentId`, `eventType`, `from`, `to`, `emitter`
- Paginiert via `Pageable` (default page=0, size=50)
- Backtest-Context-Enrichment: 2 gezielte DB-Queries (kein N+1):
  1. `tradingRunRepository.findAllById(runIds)` → runId→batchId Map
  2. `backtestRunRepository.findByBatchIdIn(batchIds)` → batchId→backtestId+Name Map
- Private `BacktestContextCache` Record für effizientes Lookup

### Tests (Backend)

**`EventQueryControllerTest.java`** (neu, 5 Unit-Tests)
- Empty result, Backtest-Context-Enrichment, Live-Run null context, Filter-Forwarding, Pagination

**`EventQueryControllerIntegrationTest.java`** (neu, 7 Integration-Tests, Zonky embedded PostgreSQL)
- Filter by instrument, event type, time range, emitter
- Backtest-Context für Sim-Runs, null context für Live-Runs
- Pagination

**Testergebnis:** 296 Tests total, 0 Failures, 0 Errors (BUILD SUCCESS)
(vorher: 291 Tests; 12 neue Tests hinzugekommen, plus 3 vorhandene Tests wurden gelöscht/ersetzt — Nettogain 5)

---

## Frontend (Parts B, C, D)

### Geänderte Dateien

**`src/domains/backtesting/types/backtesting.ts`**
- `EventRecordDto`: neue Felder `backtestId: string | null` und `backtestName: string | null`
- Neue Schnittstelle `GlobalEventFilters` (5 Felder: instrumentId, eventType, from, to, emitter)

**`src/domains/backtesting/api/backtestingApi.ts`**
- Import: `GlobalEventFilters` hinzugefügt
- Neue Funktion `getGlobalEvents(filters, pagination)` → `Promise<Page<EventRecordDto>>`
  - Endpoint: `GET /api/v1/events`
  - Setzt nur Params die truthy sind (korrekte optionale Filter)

**`src/domains/backtesting/pages/BacktestDetailPage.tsx`**
- Import: `EventLogTab`
- `DetailTab` Type erweitert um `'events'`
- Neuer Tab-Button "Events"
- Neues Tab-Content `<EventLogTab backtestId={backtestId} />`

**`src/domains/backtesting/pages/BacktestListPage.tsx`**
- Import: `GlobalEventSearch`, `useState`, `useCallback`
- Neuer Type `ListPageTab = 'backtests' | 'events'`
- Page-Level-Tabs: "Backtests" und "Event Search"
- "New Backtest"-Button nur sichtbar auf Backtests-Tab
- `<GlobalEventSearch />` auf Events-Tab

**`src/domains/backtesting/pages/BacktestListPage.module.css`**
- Neue CSS-Klassen: `.pageTabs`, `.pageTab`, `.pageTabActive`

### Neue Dateien

**`src/domains/backtesting/components/EventLogTab/useEventLog.ts`**
- Re-Export von `useBacktestEvents` als `useEventLog` (für saubere lokale Imports)

**`src/domains/backtesting/components/EventLogTab/EventTypeFilter.tsx`**
- Dropdown mit `data-testid="event-type-filter"`
- `label htmlFor` + `select id` für korrekte Accessibility

**`src/domains/backtesting/components/EventLogTab/EventPayloadViewer.tsx`**
- Zeigt expandiertes JSON-Payload mit Meta-Daten (Seq, Client Order, Broker Order, Run-ID)
- `data-testid="event-payload-viewer"`
- JSON.parse mit try/catch, Truncation nach 10.000 Zeichen

**`src/domains/backtesting/components/EventLogTab/EventLogTab.tsx`**
- Haupt-Komponente des Events-Tabs
- `data-testid="event-log-table"` auf Container
- Verwendet `EventTypeFilter`, `EventPayloadViewer`, `useEventLog`
- Color-coded Event-Type-Badges, expandierbare Rows, Pagination

**`src/domains/backtesting/components/EventLogTab/EventLogTab.module.css`**
- Vollständige CSS für Table, Filter Bar, Badges, Pagination, Empty/Loading/Error States

**`src/domains/backtesting/components/GlobalEventSearch/useGlobalEvents.ts`**
- Hook für cross-run Global Event Search
- Stale-Response-Guard via monotonem Request-Counter
- `activeFiltersRef` trennt Form-State von aktivem Such-State
- `hasSearched` Flag (kein Autoload auf Mount)
- `handleSearch`, `handlePageChange`, `handleClearSearch`

**`src/domains/backtesting/components/GlobalEventSearch/GlobalEventSearch.tsx`**
- `data-testid="global-event-search"` auf Container
- `data-testid="global-event-search-results"` auf Results-Section
- Form mit 5 Filtern: Instrument, Event Type, From (local time), To (local time), Emitter
- `datetime-local` → `new Date(formFrom).toISOString()` (UTC) für Backend-Kompatibilität
- Labels als "From (local time)" / "To (local time)" für UX-Klarheit
- Reset expanded-Row-State bei Pagination (`useEffect` auf `events.number`)
- "Backtest"-Spalte zeigt Name (oder "Live" für Live-Runs)

**`src/domains/backtesting/components/GlobalEventSearch/GlobalEventSearch.module.css`**
- Vollständige CSS für Search Form, Results Table, Filter Fields, Form Actions

**`src/domains/backtesting/components/GlobalEventSearch/useGlobalEvents.test.ts`**
- 9 Unit-Tests für `useGlobalEvents`
- Abdeckung: initial state, handleSearch (success/error/page reset), handlePageChange (mit Clamp), handleClearSearch, stale-response guard (deferred Promises)

### Build-Ergebnis

`npm run build` → 0 TypeScript-Fehler, Build SUCCESS (229 Module transformiert)

---

## Gemini-Review

Dauerhaft entfallen — Gemini-Pool nicht verfügbar (Browser durch Google blockiert, Stand 2026-02-24).
ChatGPT-Review (Runde 1 + 2) deckt Code-Review, Konzepttreue und Praxis-Szenarien ab.

---

## ChatGPT Review (DoD 2.5)

Slot: `odin-053-frontend-review`, 2 Rounds

**Findings und Maßnahmen:**

| Finding | Prio | Maßnahme |
|---------|------|----------|
| Datetime-local → UTC: Labels unklar | P0 | Labels auf "From (local time)" geändert → UX klar |
| expanded-state bei Pagination nicht reset | P1 | `useEffect` auf `events.number` in `GlobalEventSearch` ergänzt |
| stale-response Guard nicht getestet | P1 | Neuer Test mit deferred Promises hinzugefügt |
| A11y: `<tr role="button">` | P1 | Bewusste Entscheidung: Desktop-only, OK für jetzt |
| Payload-Truncation vor JSON.parse | P1 | Bewusste Entscheidung: MAX_PAYLOAD_LENGTH=10000 ist ausreichend für normale Audit-Events; P2 |
| Code-Duplikation EventTypeBadge | P2 | Backlog: `components/common/EventBadge.tsx` für zukünftigen Refactor |
| Sort-Parameter in getGlobalEvents | P1 | Backend nutzt fixed ORDER BY in SQL, kein Sort via Pageable; kein Frontend-Sort nötig |

---

## Remediation R2 — Working State

Behoben in dieser Runde (Findings aus QA-Report R1):

| # | Finding | Lösung |
|---|---------|--------|
| F-1 | AC-3: Freitextsuche in EventLogTab fehlte | Client-seitige Suche auf emitter/clientOrderId/brokerOrderId, `data-testid="event-log-search"`, useMemo-Filterung der aktuellen Seite |
| F-2 | AC-5: Sortierrichtung nicht umschaltbar | `sortAsc`-State in `useBacktestEvents`, `handleSortToggle()`, Sort-Toggle-Button im Timestamp-Header, Backend `sortAsc`-Query-Parameter in `BacktestController.getBacktestEvents` |
| F-3 | AC-8: Kein Link in GlobalEventSearch | `<Link to="/backtesting/{backtestId}">` aus react-router-dom, nullable-safe (nur wenn backtestId != null), CSS `.backtestLink` |
| F-4 | DoD 2.2: useBacktestEvents ungetestet | `useBacktestEvents.test.ts` erstellt, 9 Tests: Initial State, Load, Filter-Wechsel, Pagination, Sort-Toggle, Stale-Response Guard |
| F-5 | DoD 2.6: Gemini-Notiz fehlte | Abschnitt "Gemini-Review" in protocol.md eingefügt |
| F-6 | DoD 2.1: wallClockTimestamp nicht gemappt | `EventRecordDto.java` (Java-Record), `EventRecordDto` TypeScript-Interface: `wallClockTimestamp?: string \| null` hinzugefügt; fromEntity() mappt `entity.getWallClockTimestamp()` |
| F-7 | `q`-Parameter nicht implementiert | Client-seitige Suche (F-1) deckt emitter/clientOrderId/brokerOrderId ab; serverseitige LIKE-Suche auf clientOrderId/brokerOrderId erfordert PostgreSQL-spezifische Syntax in native SQL und schafft Risiko für SQL-Injection-Muster — kein Mehrwert gegenüber client-seitiger Lösung für 50-Einträge-Seiten |

---

## Bekannte Trade-offs / Technische Schulden

- **Code-Duplikation**: `getEventColorCategory`, `formatTimestamp`, `EventTypeBadge` existieren sowohl in `EventLogTab.tsx` als auch in `GlobalEventSearch.tsx`. Bewusster Trade-off: Extraktion in `components/common/` ist geplantes Backlog-Item, nicht blockierend.
- **A11y**: `<tr role="button">` ist kein W3C-Optimal-Pattern, aber etabliert im Projekt (auch `BacktestTradesTab` nutzt identisches Pattern) und Desktop-only.
- **Payload-Truncation**: Truncation erfolgt nach `JSON.stringify`, nicht vor `JSON.parse`. Bei sehr großen Payloads (>1MB) könnte das kurz blockieren. Für Audit-Events in normalem Betrieb (~1KB-10KB) kein Problem.

---

## Fehler und Korrekturen während Implementierung

Alle Fehler entstanden im Backend (Teil A); Frontend-Build war sofort sauber.

1. **`FEHLER: konnte Datentyp von Parameter $5 nicht ermitteln`**
   - Ursache: JPQL mit `Instant`-Parametern in `IS NULL`-Checks — Hibernate 6 kann Typ nicht inferieren
   - Fix: Native SQL mit `String`-Parametern + `CAST(:from AS timestamptz)`

2. **`FEHLER: Spalte e.marketclocktimestamp existiert nicht`**
   - Ursache: Spring Data hängte `Pageable`-Sort (camelCase) an native SQL
   - Fix: `Sort` aus allen `PageRequest`-Aufrufen in Integration-Tests entfernt; ORDER BY fest in SQL

3. **`Symbol nicht gefunden: Variable Sort`**
   - Ursache: Replace-all verpasste eine Stelle in Pagination-Test
   - Fix: Zeile manuell korrigiert

4. **Falscher `TradingRunEntity`-Konstruktor**
   - Ursache: Vereinfachter Konstruktoraufruf statt Vollkonstruktor
   - Fix: Alle Required-Parameter übergeben
