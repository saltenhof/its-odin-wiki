# ODIN-051: Agent-Optimized E2E Diagnostic Infrastructure — Implementierungsprotokoll

**Story:** ODIN-051 — Agent-Optimized E2E Diagnostic Infrastructure
**Datum:** 2026-02-24
**Status:** Implementiert, Reviews abgeschlossen, Vite-Neustart ausstehend

---

## Deliverables

| # | Deliverable | Status | Datei |
|---|-------------|--------|-------|
| 1 | `agentDiagnostics.ts` — Kern-Diagnostik-Modul | Fertig | `src/shared/diagnostics/agentDiagnostics.ts` |
| 2 | `DiagnosticsProvider.tsx` — React Context Provider | Fertig | `src/shared/diagnostics/DiagnosticsProvider.tsx` |
| 3 | `ErrorBoundary.tsx` erweitert | Fertig | `src/shared/components/ErrorBoundary.tsx` |
| 4 | `data-testid` auf allen Key-Komponenten | Fertig | diverse |
| 5 | `e2e/helpers/diagnostics.ts` — Playwright Helper | Fertig | `e2e/helpers/diagnostics.ts` |
| 6 | `e2e/live-dashboard.spec.ts` | Fertig | `e2e/live-dashboard.spec.ts` |
| 7 | `e2e/controls.spec.ts` | Fertig | `e2e/controls.spec.ts` |
| 8 | `e2e/alerts.spec.ts` | Fertig | `e2e/alerts.spec.ts` |

---

## Implementierung

### Kern-Architektur

```
window.__odinDiagnostics  (DEV only)
    ├── consoleLog[]         — console.error/warn/log Einträge
    ├── networkLog[]         — alle fetch()-Requests mit Status/Dauer
    ├── sseEvents[]          — SSE Events (via recordSseEvent())
    ├── reactErrors[]        — ErrorBoundary.componentDidCatch()
    ├── getStoreSnapshot()   — Zustand Store Snapshot (via injectStoreGetter)
    └── reset()              — Buffer leeren (zwischen Tests)
```

**DEV-Guard:** Alle Diagnostik-Funktionalität ist hinter `import.meta.env.DEV === true` geführt.
In Production sind alle Exporte No-Ops, `window.__odinDiagnostics` wird nie gesetzt.

### Hinzugefügte `data-testid` Attribute

| Komponente | data-testid |
|------------|-------------|
| Header Connection Indicator | `sse-connection-status` + `data-status={connectionStatus}` |
| ControlsPanel — Kill Switch Button | `kill-switch-button` |
| ControlsPanel — Safe Mode Button | `safe-mode-button` |
| ControlsPanel — Disconnect Banner | `controls-disabled-banner` |
| ControlsPanel — System Mode Badge | `system-mode-badge` |
| ConfirmDialog (Container) | `confirm-dialog` |
| ConfirmDialog (Cancel Button) | `confirm-dialog-cancel` |
| ConfirmDialog (Confirm Button) | `confirm-dialog-confirm` |
| TradingAlertFeed (Card) | `alert-feed` |
| TradingAlertFeed (Items) | `alert-item-{id}` |
| TradingAlertFeed (Filter Buttons) | `alert-filter-{level}` |
| AlertFeed (Card, Dashboard) | `alert-feed` |
| AlertFeed (Items, Dashboard) | `alert-item-{id}` |
| AlertFeed (Filter Buttons, Dashboard) | `alert-filter-{level}` |
| SystemHealthCard | `system-health-card` |
| PipelineStateCard | `pipeline-card-{instrumentId}` |
| PositionPanel | `position-panel` |
| ConnectionBanner | `sse-connection-banner` |
| ToasterNotification | `toaster-notification` |
| PipelineDetailPage (Badge) | `pipeline-state-badge` |
| PipelineDetailPage (Grid) | `realtime-decisions` |

### Modifizierte Dateien

- `src/app/App.tsx` — DiagnosticsProvider Wrapper
- `src/shared/components/ErrorBoundary.tsx` — `recordReactError()` in `componentDidCatch`
- `src/shared/components/Card.tsx` — `'data-testid'?: string` Prop
- `src/shared/components/Badge.tsx` — `'data-testid'?: string` Prop
- `src/shared/components/ConfirmDialog.tsx` — data-testid auf Dialog und Buttons
- `src/shared/components/ToasterNotification.tsx` — data-testid auf Wrapper
- `src/shared/components/ConnectionBanner.tsx` — data-testid auf Banner
- `src/app/Header.tsx` — data-testid + data-status auf Connection Indicator
- `src/domains/trading-operations/components/SystemHealthCard.tsx` — data-testid auf Card
- `src/domains/trading-operations/components/PipelineStateCard.tsx` — data-testid auf Container
- `src/domains/trading-operations/components/SystemModeBadge.tsx` — `'data-testid'?: string` Prop
- `src/domains/trading-operations/components/ControlsPanel.tsx` — data-testid auf Buttons/Banner/Badge
- `src/domains/trading-operations/components/TradingAlertFeed.tsx` — data-testid auf Feed/Items/Filters
- `src/domains/dashboard/components/AlertFeed.tsx` — data-testid auf Feed/Items/Filters
- `src/domains/trading-operations/components/PositionPanel.tsx` — data-testid auf Card
- `src/domains/trading-operations/pages/PipelineDetailPage.tsx` — data-testid auf Badge und Grid
- `vite.config.ts` — Proxy `/api` und `/actuator` auf localhost:3300

---

## Test-Verifikation

### Port 3001 Ergebnis (frischer Vite-Start, vor Context-Kompaktierung)

```
Running 11 tests using 1 worker
ok 1  alerts.spec.ts:31 › Alert Feed shows empty state when no alerts
ok 2  alerts.spec.ts:61 › Alert Filter Buttons are visible
ok 3  alerts.spec.ts:78 › Alert filter tabs are interactive
ok 4  controls.spec.ts:35 › Kill-Switch button is visible
skipped controls.spec.ts:48 › Kill-Switch button opens Confirm Dialog (SSE not connected)
skipped controls.spec.ts:95 › Cancel button closes the Confirm Dialog without API call (SSE not connected)
ok 7  controls.spec.ts:152 › System-Mode-Badge shows a valid mode text
ok 8  live-dashboard.spec.ts:30 › Dashboard loads without console or network errors
ok 9  live-dashboard.spec.ts:60 › SystemHealthCard is visible and contains text
ok 10 live-dashboard.spec.ts:86 › Alert Feed container is reachable
ok 11 live-dashboard.spec.ts:101 › Controls panel shows system mode badge
```

### Port 3000 Status (nach Context-Kompaktierung)

Der Vite Dev Server auf Port 3000 hat während HMR (nach Hinzufügen des `DiagnosticsProvider` in `App.tsx`)
seinen esbuild-Service verloren. Der Server serviert gecachte Versionen alter Dateien für bekannte Routen
(`/backtesting` funktioniert), kann aber neue Dateien und Routen (`/trading`) nicht kompilieren.

**Symptom:** `[plugin:vite:esbuild] The service is no longer running` für `TradingOperationsPage.tsx`

**Lösung:** Vite Dev Server neu starten:
```bash
# Im its-odin-ui Verzeichnis, bestehenden Vite-Prozess beenden und neu starten:
npx vite
```

Nach dem Neustart alle Tests laufen lassen:
```bash
npx playwright test e2e/alerts.spec.ts e2e/controls.spec.ts e2e/live-dashboard.spec.ts e2e/backtest-creation.spec.ts --reporter=list
```

---

## ChatGPT Reviews

### Runde 1 — Bugs & Robustheit, API-Design, Fetch-Interception, Store-Snapshot

**Kritische Findings (alle umgesetzt):**

1. **HMR Double-Install:** Modul-Singleton wird bei HMR zurückgesetzt, während `window.__odinDiagnostics` noch existiert → doppeltes Wrapping von console/fetch.
   - Fix: `getOrInstallDiagnostics()` prüft nun `window.__odinDiagnostics !== undefined` bevor neu installiert wird.

2. **Fetch Method Detection für `fetch(Request)`:** Bei `fetch(new Request(url, { method: 'POST' }))` ohne `init` wurde fälschlich `GET` geloggt.
   - Fix: `init?.method ?? (input instanceof Request ? input.method : undefined) ?? 'GET'`

3. **`status: 0` nicht als Fehler gewertet:** Netzwerk-Level-Fehler (CORS, Offline) hatten status=0 und wurden im Filter `>= 400` übersehen.
   - Fix: Filter ist jetzt `status === 0 || status >= 400` (sowohl in `agentDiagnostics.ts` als auch in `collectDiagnostics`)

4. **`reactErrors` nicht gekappt:** Memory-Growth bei langen Test-Sessions.
   - Fix: `clampLog(reactErrors)` nach `push` in `recordReactError()`

5. **SSE Inline-Konstante:** `recordSseEvent()` nutzte `500` statt `MAX_LOG_ENTRIES`.
   - Fix: Jetzt `clampLog(diag.sseEvents)` — konsistente Nutzung der Konstante

**Wichtige Findings (aufgeschoben für spätere Story):**
- `readonly ConsoleEntry[]` verhindert nicht externes Mutieren — `ReadonlyArray<>` wäre korrekter
- `_setStoreGetter` auf Window sichtbar — `Object.defineProperty` mit `enumerable: false` wäre besser
- `unhandledrejection` und `window.onerror` Events nicht erfasst
- Schema-Versionierung für stabile Agent-Interpretation fehlt

### Runde 2 — E2E Test Specs (Robustheit, SSE-Handling, Agent-UX, Abdeckung)

**Kritische Findings (alle umgesetzt):**

1. **Early `return` ≠ Test-Skip:** `return` in einer Test-Funktion zählt als "passed", nicht als "skipped".
   - Fix: `test.skip(true, 'SSE not connected — ...')` in controls.spec.ts für beide Dialog-Tests

2. **`waitForSseConnected()` Helper:** Statt einfachem `getAttribute` nun `expect.poll()` für zuverlässiges Warten.
   - Fix: Neuer `waitForSseConnected(page, timeoutMs)` Helper in controls.spec.ts

3. **`waitForTimeout(200)` nach Filter-Klick:** Flaky Sleep statt Observable-Assertion.
   - Fix: `await expect(filterButton).toHaveAttribute('aria-pressed', 'true')` nach jedem Klick

4. **`waitForTimeout(2_000)` in live-dashboard:** Sleep durch deterministisches Warten ersetzt.
   - Fix: `expect.poll()` auf `data-status` bis Terminal-State erreicht

**Aufgeschobene Findings:**
- `afterEach` Diagnostics Attachment (automatisch immer sammeln)
- `test.step()` für agent-freundliche Timeline
- Request-Body Capture für POST-Fehler
- Mock-Backend / MSW für deterministische SSE-Tests
- Coverage: Confirm-Pfad (POST /controls/kill), Fehlerpfade, Filter-Wirkung (echte Filterung assertieren)

---

## Gemini Review

### Dimension 1: Bugs & Robustheit

Übereinstimmende Findings mit ChatGPT (alle umgesetzt):
- HMR Double-Install — behoben
- Store Snapshot Serialisierung — potentiell unsicher, aber Typing schützt bei aktuellen Store-Strukturen

### Dimension 2: Agent-UX

Wesentliche Findings:
- `takeAnnotatedScreenshot` ist nicht wirklich "annotiert" (kein Overlay) — Umbenennung oder echte Annotation empfohlen
- Request-Body fehlt für POST-Fehler-Debugging
- Business-Fehler (falsche Store-Werte) werden nicht automatisch erkannt

### Dimension 3: Wartbarkeit

Kritisches Finding (umgesetzt):
- **Silent False-Positive:** Wenn `window.__odinDiagnostics` fehlt, liefert `collectDiagnostics()` leere Arrays → `expectNoErrors()` ist grün obwohl nichts geprüft wurde.
  - Fix: `console.warn()` im Browser-Kontext wenn Diagnostics-Objekt fehlt

- **Store-Mapping Kopplung:** `DevDiagnosticsProvider` listet explizit Store-Felder auf → bei Store-Refactors muss Provider manuell angepasst werden.
  - Status: Bewusste Design-Entscheidung, bekanntes Maintenance-Risiko, dokumentiert

---

## Offene Punkte (für zukünftige Stories)

| Punkt | Priorität | Begründung |
|-------|-----------|------------|
| `ReadonlyArray<>` für AgentDiagnostics-Interface | LOW | Defensive, kein Bug |
| `_setStoreGetter` als non-enumerable | LOW | Sauberkeit |
| `unhandledrejection` + `window.onerror` Capture | MEDIUM | Fehler ohne React-Boundary werden nicht erfasst |
| Schema-Versionierung (`schemaVersion: 1`) | LOW | Agent-Stabilitätsnetz |
| `afterEach` Diagnostics Attachment | MEDIUM | Automatisches Sammeln ohne expliziten Test-Aufruf |
| Mock-Backend / MSW für deterministische SSE Tests | MEDIUM | Eliminiert Skip-Pattern für Dialog-Tests |
| `takeAnnotatedScreenshot` echte Annotation (DOM-Overlay) | LOW | Nice-to-have für Agent-Inspection |
| Request-Body Capture für POST-Fehler | LOW | Debug-Mehrwert |
| Confirm-Pfad Test (POST /controls/kill) | HIGH | Wichtigster Controls-Flow fehlt |
| Filter-Wirkung assertieren (nicht nur Klick-Response) | MEDIUM | Echte Funktionalität testen |

---

## Nicht-Commit

Kein Commit wurde erstellt. Die Story ist implementiert und reviewed.
Der User kann nach Vite-Neustart die Tests verifizieren und dann committen.
