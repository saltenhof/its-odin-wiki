# QA Report — ODIN-053 (Runde 2)

**Story:** Backtest Event Log UI: Audit-Analyse & Laufübergreifende Auswertung
**Prüfdatum:** 2026-02-24
**Commits Backend:** 582fea7 | **Commits Frontend:** baf026c + 28fb415
**Bekannte Test-Erfolge:** Backend 296 Tests (0 Failures), Frontend 268 Tests (0 Failures)

---

## 1. R1-Findings: Nachverfolgung

### F-1 (AC-3): Freitextsuche in EventLogTab

**Status: BEHOBEN**

`EventLogTab.tsx` enthält:
- `<input ... data-testid="event-log-search" ...>` (Zeile 283)
- `useMemo`-basierte Filterung auf `event.emitter`, `event.clientOrderId`, `event.brokerOrderId` (Zeilen 251–264)
- Reset bei Filter-Wechsel: `useEffect` setzt `searchQuery` zurück bei `selectedEventType`-Änderung (Zeilen 237–239)

Beispiel-Filterlogik:
```typescript
const matchesEmitter = event.emitter.toLowerCase().includes(query);
const matchesClientOrderId =
  event.clientOrderId != null && event.clientOrderId.toLowerCase().includes(query);
const matchesBrokerOrderId =
  event.brokerOrderId != null && event.brokerOrderId.toLowerCase().includes(query);
return matchesEmitter || matchesClientOrderId || matchesBrokerOrderId;
```

**PASS**

---

### F-2 (AC-5): Sort-Toggle

**Status: BEHOBEN**

- `useBacktestEvents.ts`: `sortAsc: boolean` State (Zeile 54), `handleSortToggle()` (Zeilen 124–127)
- `backtestingApi.ts`: `sortAsc`-Parameter an Backend (Zeile 189: `params.set('sortAsc', String(sortAsc))`)
- `BacktestController.java`: `@RequestParam(defaultValue = "true") boolean sortAsc` in `getBacktestEvents` (Zeile 358); `Sort.Direction direction = sortAsc ? Sort.Direction.ASC : Sort.Direction.DESC;` (Zeile 382)
- `EventLogTab.tsx`: Sort-Button im Timestamp-Header mit `{sortAsc ? ' ↑' : ' ↓'}` Indikator (Zeilen 308–319)

**PASS**

---

### F-3 (AC-8): Link zum Backtest-Lauf in GlobalEventSearch

**Status: BEHOBEN**

`GlobalEventSearch.tsx` (Zeilen 178–189):
```tsx
{event.backtestId != null && event.backtestName != null ? (
  <Link
    to={`/backtesting/${event.backtestId}`}
    className={styles.backtestLink ?? ''}
    title={`Open backtest: ${event.backtestId}`}
    onClick={(e) => e.stopPropagation()}
  >
    {event.backtestName}
  </Link>
) : (
  <span className={styles.backtestNameNone ?? ''}>Live</span>
)}
```

Link ist null-safe (nur wenn `backtestId != null && backtestName != null`), verwendet `react-router-dom`s `<Link>`, stoppt Row-Toggle per `e.stopPropagation()`.

**PASS**

---

### F-4 (DoD 2.2): Unit-Tests für useBacktestEvents

**Status: BEHOBEN**

`useBacktestEvents.test.ts` enthält 11 Tests in 6 Describe-Blöcken:
- Initial State (2 Tests): leerer State, `sortAsc=true`; API-Call on mount
- eventType filter change (1 Test): Page-Reset + API-Call mit eventType
- Pagination (2 Tests): Page-Wechsel, Clamp negativer Seiten auf 0
- Sort toggle (2 Tests): Toggle-Richtung, Doppel-Toggle zurück auf ASC
- Error handling (1 Test): Error-State bei fehlgeschlagenem API-Call
- eventTypes fetch (2 Tests): Füllung aus API, Fallback auf leer
- Stale-response guard (1 Test): Später aufgelöster erster Request wird ignoriert

**PASS**

---

### F-5 (DoD 2.6): Gemini-Review-Notiz in protocol.md

**Status: BEHOBEN**

`protocol.md` enthält Abschnitt `## Gemini-Review` mit Inhalt:
> "Dauerhaft entfallen — Gemini-Pool nicht verfügbar (Browser durch Google blockiert, Stand 2026-02-24). ChatGPT-Review (Runde 1 + 2) deckt Code-Review, Konzepttreue und Praxis-Szenarien ab."

**PASS**

---

### F-6 (DoD 2.1): wallClockTimestamp nicht gemappt

**Status: BEHOBEN**

- `EventRecordDto.java` (Zeile 42): `Instant wallClockTimestamp` als Record-Komponente; `fromEntity()` mappt `entity.getWallClockTimestamp()` (Zeile 83)
- `backtesting.ts` (Zeile 269): `readonly wallClockTimestamp: string | null;`

JavaDoc auf Record-Parameter vorhanden: `@param wallClockTimestamp wall-clock timestamp when the event was recorded, or null`.

**PASS**

---

### F-7: Fehlender q-Parameter (Backend)

**Status: DOKUMENTIERT UND AKZEPTIERT**

`protocol.md` (Abschnitt "Remediation R2") enthält explizite Begründung:
> "Client-seitige Suche (F-1) deckt emitter/clientOrderId/brokerOrderId ab; serverseitige LIKE-Suche auf clientOrderId/brokerOrderId erfordert PostgreSQL-spezifische Syntax in native SQL und schafft Risiko für SQL-Injection-Muster — kein Mehrwert gegenüber client-seitiger Lösung für 50-Einträge-Seiten"

Die Entscheidung ist fachlich begründet. AC-3 ist über die client-seitige Implementierung (F-1) erfüllt. Der Story-Scope lässt ausdrücklich offen: "Client-seitig ist okay für Seiten bis 50 Einträge."

**PASS (Akzeptiert)**

---

## 2. Vollständige AC-Prüfung (AC-1 bis AC-9)

### AC-1: Tab "Events" in BacktestDetailPage

**PASS** — Unverändert gegenüber R1. `DetailTab` enthält `'events'`, Tab-Button und `<EventLogTab backtestId={backtestId} />` korrekt integriert.

---

### AC-2: Event-Typ-Dropdown dynamisch befüllt

**PASS** — Unverändert gegenüber R1. `useBacktestEvents` ruft `api.getBacktestEventTypes(backtestId)` auf; `EventTypeFilter` rendert dynamisch.

---

### AC-3: Freitextsuche vorhanden (vorher FAIL)

**PASS** — `data-testid="event-log-search"` vorhanden. Client-seitige Filterung auf `emitter`, `clientOrderId`, `brokerOrderId` via `useMemo`. Suche reagiert sofort auf Eingabe (kein Debounce nötig bei 50 Einträgen pro Seite).

---

### AC-4: Payload-Viewer expandierbar

**PASS** — Unverändert. `EventPayloadViewer` mit Toggle, Standard eingeklappt.

---

### AC-5: Chronologische Sortierung + umschaltbar (vorher FAIL)

**PASS** — Standard: ASC (Backend `sortAsc=true`). Sort-Toggle-Button im Timestamp-Header schaltet zwischen ASC (↑) und DESC (↓) um. `handleSortToggle()` in `useBacktestEvents` invertiert `sortAsc` und setzt Page auf 0 zurück.

---

### AC-6: GET /api/v1/events filterbar

**PASS** — Unverändert. Endpoint mit 5 optionalen Filtern (`instrumentId`, `eventType`, `from`, `to`, `emitter`) + Pagination. `q`-Parameter bewusst nicht implementiert (Begründung: F-7).

---

### AC-7: Cross-Run-Ansicht in UI vorhanden

**PASS** — Unverändert. `GlobalEventSearch` als "Event Search"-Tab in `BacktestListPage`.

---

### AC-8: Link zum Backtest-Lauf in Cross-Run-Ansicht (vorher FAIL)

**PASS** — `<Link to="/backtesting/${event.backtestId}">` implementiert. Null-safe (Live-Runs zeigen "Live"). `e.stopPropagation()` verhindert ungewolltes Row-Toggle beim Klick.

---

### AC-9: data-testid Attribute

**PASS** — Alle 5 geforderten Attribute vorhanden:

| Attribut | Datei | Zeile |
|----------|-------|-------|
| `event-log-table` | `EventLogTab.tsx` | 269 |
| `event-type-filter` | `EventTypeFilter.tsx` | (bestehend) |
| `event-payload-viewer` | `EventPayloadViewer.tsx` | (bestehend) |
| `global-event-search` | `GlobalEventSearch.tsx` | 273 |
| `global-event-search-results` | `GlobalEventSearch.tsx` | 380 |

Neu in R2: `event-log-search` auf dem Freitext-Suchfeld (Zeile 283, `EventLogTab.tsx`).

---

## 3. Spot-Check Code-Qualität

### 3.1 Kein `any` (Frontend)

| Datei | Ergebnis |
|-------|----------|
| `EventLogTab.tsx` | PASS — kein `any` |
| `useBacktestEvents.ts` | PASS — kein `any` |
| `useBacktestEvents.test.ts` | PASS — Mock-Parameter als `unknown[]` |
| `GlobalEventSearch.tsx` | PASS — kein `any` |
| `backtestingApi.ts` | PASS — kein `any` |
| `backtesting.ts` | PASS — `payload: string`, kein `any` |

### 3.2 CSS Modules

Alle neuen Dateien verwenden ausschließlich `styles.*`-Referenzen. Keine Inline-Styles.

### 3.3 Feature→Feature-Import

- `GlobalEventSearch.tsx` importiert `EventPayloadViewer` aus `../EventLogTab/` — beides liegt in `src/domains/backtesting/`. Kein Cross-Feature-Import.
- Keine Imports aus `dashboard`, `trading-operations` oder anderen Domains.

**PASS**

### 3.4 JavaDoc auf neuen Backend-Klassen

| Element | JavaDoc |
|---------|---------|
| `EventRecordDto` (Record-Klasse) | PASS — Klassen-JavaDoc + alle `@param`-Einträge inkl. neu hinzugefügtem `wallClockTimestamp` |
| `EventRecordDto.fromEntity(entity)` | PASS |
| `EventRecordDto.fromEntity(entity, backtestId, backtestName)` | PASS |
| `BacktestController.getBacktestEvents` | PASS — `@param sortAsc` dokumentiert |

**PASS**

---

## 4. Gesamtbewertung je DoD und AC

| Punkt | R1-Status | R2-Status |
|-------|-----------|-----------|
| DoD 2.1 Code-Qualität (Backend) | PASS | PASS |
| DoD 2.1 Code-Qualität (Frontend) | PASS | PASS |
| DoD 2.1 wallClockTimestamp gemappt | FAIL (F-6) | **PASS** |
| DoD 2.2 Unit-Tests Frontend (useBacktestEvents) | FAIL (F-4) | **PASS** |
| DoD 2.3 Integrationstests | PASS | PASS |
| DoD 2.5 ChatGPT-Sparring | PASS | PASS |
| DoD 2.6 Gemini-Notiz in protocol.md | FAIL (F-5) | **PASS** |
| DoD 2.7 Protokolldatei | PASS | PASS |
| AC-1 Tab "Events" | PASS | PASS |
| AC-2 Dropdown dynamisch | PASS | PASS |
| AC-3 Freitextsuche | FAIL (F-1) | **PASS** |
| AC-4 Payload-Viewer | PASS | PASS |
| AC-5 Sort umschaltbar | FAIL (F-2) | **PASS** |
| AC-6 GET /api/v1/events | PASS | PASS |
| AC-7 Cross-Run-Ansicht | PASS | PASS |
| AC-8 Link zum Backtest-Lauf | FAIL (F-3) | **PASS** |
| AC-9 data-testid Attribute | PASS | PASS |

**Alle 7 R1-Findings behoben. Keine neuen Findings.**

---

QS-SIGNAL: PASS
