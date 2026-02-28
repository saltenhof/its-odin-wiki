# Story: ODIN-053 — Backtest Event Log UI: Audit-Analyse & Laufübergreifende Auswertung

## Metadaten

| Feld | Wert |
|------|------|
| **Titel** | Backtest Event Log UI: Audit-Analyse & Laufübergreifende Auswertung |
| **Modul** | odin-frontend (primär), odin-backtest (Backend-Erweiterung für Cross-Run-Endpoint) |
| **Phase** | 1 |
| **Abhängigkeiten** | Keine (kann unabhängig von ODIN-052 entwickelt werden; Testdaten können aus beliebigem bestehenden Backtest-Lauf stammen) |
| **Geschätzter Umfang** | L |

---

## Kontext

ODIN verfügt über ein vollständiges Audit-System mit Hash-Chain-Integrität, HMAC-signierten Trade-Intents und einem strukturierten `EventRecord`-Modell (odin-audit). Jedes relevante Systemereignis (State-Changes, Order-Submissions, Fills, Stop-Anpassungen, Risk-Gate-Entscheidungen) wird persistent gespeichert. Diese Daten sind bisher nur über die Backend-API abrufbar, aber nicht in der UI zugänglich.

Backend-Endpoints existieren bereits:
- `GET /api/v1/backtests/{id}/events?eventType=...` (paginiert, Filter nach Event-Typ)
- `GET /api/v1/backtests/{id}/event-types` (distinct Event-Typen für Dropdown)

Ziel dieser Story: Eine vollwertige Event-Log-UI in der Backtest-Detail-Ansicht implementieren, plus einen neuen Backend-Endpoint für laufübergreifende Event-Abfragen, und eine entsprechende Cross-Run-Ansicht im Frontend.

---

## Scope

**In Scope:**

*Frontend — Backtest-Detail-Ansicht:*
- Neuer Tab "Events" in `BacktestDetailPage` (neben Summary, Tage, Chart)
- Tabellarische Darstellung der EventRecords mit Spalten: Zeitstempel, Event-Typ, Emitter, Instrument, Sequenznummer, Client-Order-ID (nullable), Payload (expandierbar)
- Filter: Dropdown nach Event-Typ (befüllt aus `/event-types`-Endpoint)
- Freitextsuche auf den Feldern `emitter`, `clientOrderId`, `brokerOrderId`
- Pagination (serverseitig, 50 Einträge pro Seite)
- Payload-Viewer: JSONB-Payload expandierbar/einklappbar, formatiert als JSON-Tree oder Syntax-Highlighted
- Chronologische Sortierung (Standard: marketClockTimestamp ASC)

*Frontend — Cross-Run-Ansicht:*
- Neue Seite oder dedizierter Bereich in der Backtest-Liste (`BacktestHistoryPage`): "Globale Event-Suche"
- Filter: Instrument, Event-Typ, Datumsbereich (marketClockTimestamp)
- Freitextsuche auf `emitter`, `clientOrderId`, `eventType`
- Zeigt Ergebnisse aus ALLEN Backtest-Läufen (oder ausgewählten)
- Jede Zeile verlinkt zurück zum zugehörigen Backtest-Lauf

*Backend — neuer Cross-Run-Endpoint:*
- `GET /api/v1/events?instrumentId=&eventType=&from=&to=&emitter=&page=&size=`
- Abfragebereich: alle BacktestRunEntities (und optional LiveRuns)
- Gibt `Page<EventRecordDto>` zurück
- EventRecordDto enthält zusätzlich: `backtestId`, `backtestName`, `runId`

**Out of Scope:**
- Live-Trading Event-Log (nur Backtest-Kontext in dieser Story)
- Hash-Chain-Verifikation in der UI (nur Darstellung, keine Integritätsprüfung)
- Export-Funktionalität (CSV-Download)
- Real-Time Event-Stream via SSE

---

## Akzeptanzkriterien

- [ ] **AC-1:** Der Tab "Events" in `BacktestDetailPage` zeigt alle EventRecords eines Backtestlaufs in einer paginierten Tabelle.
- [ ] **AC-2:** Der Event-Typ-Filter (Dropdown) wird dynamisch aus dem Backend befüllt — nur tatsächlich vorhandene Typen werden angezeigt.
- [ ] **AC-3:** Freitextsuche filtert die Ergebnisse sofort (Client-seitig auf aktueller Seite ODER Server-seitiger Search-Parameter).
- [ ] **AC-4:** Der Payload-Viewer ermöglicht es, den JSON-Payload eines Events zu expandieren und zu lesen.
- [ ] **AC-5:** Die Tabelle ist chronologisch sortiert (marketClockTimestamp ASC als Default); Sortierrichtung umschaltbar.
- [ ] **AC-6:** Der neue Backend-Endpoint `GET /api/v1/events` liefert Events über Backtest-Läufe hinweg und ist filterbar nach Instrument, Event-Typ und Datumsbereich.
- [ ] **AC-7:** Die Cross-Run-Ansicht (globale Event-Suche) existiert in der UI und nutzt den neuen Endpoint.
- [ ] **AC-8:** Jede Event-Zeile in der Cross-Run-Ansicht enthält einen Link zum zugehörigen Backtest-Lauf.
- [ ] **AC-9:** Alle neuen UI-Komponenten haben `data-testid`-Attribute (z.B. `event-log-table`, `event-type-filter`, `event-payload-viewer`, `global-event-search`).

---

## Technische Details

### Backend: Neuer Cross-Run-Endpoint

**Modul:** odin-backtest (Controller) oder odin-audit (Query-Service)

**Neue Klassen/Methoden:**
- `EventQueryController` (neu, odin-app) oder Erweiterung von `BacktestController`:
  ```
  GET /api/v1/events
    ?instrumentId=IREN       (optional)
    &eventType=ENTRY_FILLED  (optional)
    &from=2026-02-01T00:00:00Z  (optional, ISO-8601)
    &to=2026-02-28T23:59:59Z    (optional, ISO-8601)
    &emitter=OMS             (optional)
    &q=clientOrderId         (optional, Freitext)
    &page=0&size=50
    → Page<EventRecordDto>
  ```
- `EventRecordDto` (odin-api): bestehendes DTO erweitern um `backtestId` (UUID, nullable), `backtestName` (String, nullable), `runId` (UUID)
- `EventRecordRepository`: neue Query-Methode `findByFilters(...)` mit optionalen Parametern

**Abfragestrategie:**
- Join `event_record` → `trading_run` → `backtest_run` (über `batchId`)
- JPQL oder Native SQL mit dynamischen Filtern (Spring Data Specifications oder @Query mit optionalen Parameter-Checks)
- Paginierung über `Pageable`

### Frontend: Event-Log-Tab

**Neue Dateien:**
- `src/features/backtest/components/EventLogTab/EventLogTab.tsx`
- `src/features/backtest/components/EventLogTab/useEventLog.ts` (Custom Hook: API-Calls)
- `src/features/backtest/components/EventLogTab/EventPayloadViewer.tsx` (JSON-Expandable)
- `src/features/backtest/components/EventLogTab/EventTypeFilter.tsx`

**State:**
- Aktuell gewählter Event-Typ-Filter (string | null)
- Freitext-Query (string)
- Aktuelle Seite + Seitengröße
- Geladene Events (Page<EventRecordDto>)

**API-Calls:**
- `GET /api/v1/backtests/{id}/event-types` → befüllt Event-Typ-Dropdown
- `GET /api/v1/backtests/{id}/events?eventType=X&page=0&size=50` → befüllt Tabelle

### Frontend: Cross-Run Event-Suche

**Neue Dateien:**
- `src/features/backtest/components/GlobalEventSearch/GlobalEventSearch.tsx`
- `src/features/backtest/components/GlobalEventSearch/useGlobalEvents.ts`

**Integration:**
- Neuer Tab oder Abschnitt in der Backtest-Listen-Seite (`BacktestHistoryPage`)
- Formular: Instrument-Filter, Event-Typ, Datumsbereich From/To, Emitter-Freitext
- Ergebnistabelle: backtestName (als Link), instrumentId, eventType, emitter, marketClockTimestamp, payload (expandierbar)

### EventRecord-Felder (für Spaltendefinition)

```typescript
interface EventRecordDto {
  eventId: string;           // UUID
  runId: string;             // UUID
  backtestId?: string;       // neu: UUID (nullable)
  backtestName?: string;     // neu: String (nullable)
  instrumentId: string;      // "IREN"
  eventType: string;         // "ENTRY_FILLED"
  emitter: string;           // "OMS", "RulesEngine", "RiskGate"
  sequenceNumber: number;
  clientOrderId?: string;
  brokerOrderId?: string;
  marketClockTimestamp: string;  // ISO-8601
  wallClockTimestamp: string;    // ISO-8601
  payload: Record<string, unknown>;  // JSONB
}
```

### Frontend-Architektur-Regeln (strikt einhalten)

- Feature-Folder: `src/features/backtest/` — alle neuen Komponenten gehören hierhin
- Kein Feature→Feature-Import (andere Features nicht importieren)
- TypeScript strict, keine `any`-Typen
- CSS Modules für Styling

---

## Konzept-Referenzen

| Dokument | Abschnitt |
|----------|-----------|
| `docs/concept/intraday-agent-concept.md` | Kap. 19: Security & Isolation (Hash-Chain, Audit-Log, HMAC) |
| `docs/backend/architecture/08-data-model.md` | EventRecord-Entity, Zwei Schreibpfade, Flyway-Migrations |
| `docs/frontend/architecture/09-frontend.md` | Feature-basierte Ordnerstruktur, SSE/REST-Kommunikation |

---

## Guardrail-Referenzen

| Guardrail | Pfad |
|-----------|------|
| Frontend-Guardrail | `docs/frontend/guardrails/frontend.md` |
| Entwicklungsprozess | `docs/backend/guardrails/development-process.md` |
| CLAUDE.md | `T:/codebase/its_odin/CLAUDE.md` |

---

## Definition of Done

### 2.1 Code-Qualität
- [ ] Implementierung vollständig gemäß Akzeptanzkriterien
- [ ] Backend: `mvn compile -pl odin-app,odin-audit` fehlerfrei
- [ ] Frontend: `npm run build` (Vite) fehlerfrei, TypeScript strict (keine `any`)
- [ ] Kein `var` — explizite Typen (Backend)
- [ ] Keine Magic Numbers — `private static final` Konstanten (Backend)
- [ ] Records für DTOs (Backend)
- [ ] JavaDoc auf allen neuen/geänderten public Backend-Klassen und Methoden
- [ ] Keine TODO/FIXME-Kommentare
- [ ] `data-testid`-Attribute auf allen neuen UI-Komponenten

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests für neue Backend-Service-Methoden (Event-Filter-Query)
- [ ] Unit-Tests für neue Frontend-Custom-Hooks (gemockte API-Calls)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] `*IntegrationTest` für neuen `/api/v1/events`-Endpoint (mit Zonky-Embedded-DB)
- [ ] Test: Filter nach Instrument → nur zugehörige Events zurückgegeben
- [ ] Test: Filter nach Event-Typ → korrekte Filterung
- [ ] Test: Pagination → korrekte Seitenanzahl

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT nach Edge-Cases gefragt: leere Event-Logs, sehr große Payloads, fehlende batchId, Filter-Kombinationen
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini (drei Dimensionen)
- [ ] Dimension 1: Code-Review (Bugs, SQL-Injection-Risiken, Null-Safety, TypeScript-Typen)
- [ ] Dimension 2: Konzepttreue (passt die UI zum Audit-Konzept Kap. 19?)
- [ ] Dimension 3: Praxis-Review (UX-Gaps, fehlende Filter, wichtige Event-Typen die nicht dargestellt werden)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten erstellt und aktuell gehalten

### 2.8 Abschluss
- [ ] Commit mit aussagekräftiger Message
- [ ] Push auf Remote

---

## Notizen für den Implementierer

- **Payload ist JSONB** — im Frontend als `Record<string, unknown>` typen, nicht als `string` oder `any`. JSON.stringify/parse nicht nötig — Backend liefert bereits geparsten JSON.
- **Große Payloads:** Manche Payloads können sehr groß sein (z.B. LLM-Analyse-Payloads). Der Payload-Viewer soll standardmäßig eingeklappt sein (kein Auto-Expand).
- **sequenceNumber als Sort-Key:** Innerhalb eines Runs ist `sequenceNumber` der zuverlässige Sort-Key (monoton, lückenlos). Für Cross-Run-Ansicht: `marketClockTimestamp`.
- **Cross-Run-Query kann langsam sein:** Bei vielen Runs und Millionen Events muss die Query performant sein. Index auf `event_record.marketClockTimestamp` prüfen/anlegen (Flyway-Migration falls nötig).
- **backtestId/backtestName im DTO:** Diese Felder sind nullable (Events aus Live-Runs haben keine backtestId). In der Cross-Run-Ansicht: LiveRun-Events können ignoriert oder speziell markiert werden.
- **Frontend-Architektur:** Alle neuen Komponenten gehören in `src/features/backtest/`. Kein Import aus anderen Feature-Ordnern.
- **Freitextsuche:** Kann client-seitig (auf geladener Seite) oder server-seitig umgesetzt werden. Server-seitig ist skalierbar aber aufwändiger. Client-seitig ist okay für Seiten bis 50 Einträge. Entscheidung dem Implementierer überlassen.
