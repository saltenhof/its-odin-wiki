# ODIN-040: Frontend Live Trading Dashboard

**Modul:** odin-frontend (its-odin-ui)
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-037 (Structured SSE Events)
**Geschaetzter Umfang:** L

---

## Kontext

Das Frontend hat eine Trading-Operations-Seite mit Mock-Daten. Die Integration mit den echten SSE-Streams fehlt. Das Dashboard muss Echtzeit-Updates fuer alle Pipelines zeigen: Zustand, Position, P&L, Indikatoren, Alerts. ODIN-037 liefert die strukturierten Backend-Events — diese Story implementiert die Frontend-Seite der Integration.

## Scope

**In Scope:**
- SSE-Integration fuer Global-Stream (`/api/v1/stream/global`) und Per-Pipeline-Stream (`/api/v1/stream/pipeline/{instrumentId}`)
- Dashboard-Kacheln pro Pipeline: State, Position, Unrealized P&L, Regime
- Global-Kacheln: Tages-P&L, Exposure, Trade-Count, System-Health
- Alert-Feed mit Echtzeit-Updates
- Kill-Switch-Button mit Confirmation-Dialog (bereits vorhanden, integrieren)
- Reconnect-Logik mit Backoff

**Out of Scope:**
- Chart-Integration (separates Feature)
- Historische Daten im Dashboard (nur Echtzeit)
- Mobile-Responsive Design (Desktop-only)
- Dark/Light-Theme-Toggle (Dark Theme ist der einzige Modus)

## Akzeptanzkriterien

- [ ] Global-Stream verbunden und alle Event-Typen korrekt verarbeitet
- [ ] Per-Pipeline-Stream bei Navigation auf Pipeline-Detail aktiviert, bei Verlassen deaktiviert
- [ ] Dashboard zeigt pro Pipeline: State (farbcodiert nach State-Enum), Position (Qty + Avg Price), Unrealized P&L (gruen wenn positiv, rot wenn negativ)
- [ ] Global-Kacheln: Tages-P&L (gruen/rot), Gesamt-Exposure, Active Trades, System-Health-Indicator
- [ ] Alert-Feed: Neueste Alerts oben, Level-farbcodiert (INFO=blau, WARNING=gelb, CRITICAL=orange, EMERGENCY=rot)
- [ ] Kill-Switch: ConfirmDialog vor Ausfuehrung, `POST /api/v1/control/kill`, visuelles Feedback (Spinner + Ergebnis)
- [ ] Reconnect-Logik: Bei SSE-Disconnect automatische Wiederverbindung mit exponentiellem Backoff (Initial 10s, Max 60s)
- [ ] ConnectionBanner zeigt Verbindungsstatus (connecting/connected/disconnected) — kein false-positive bei initialem Aufbau

## Technische Details

**Dateien:**
- `its-odin-ui/src/domains/trading-operations/hooks/usePipelineStream.ts` (Erweiterung)
- `its-odin-ui/src/shared/hooks/useSseGlobalStream.ts` (Erweiterung)
- `its-odin-ui/src/domains/trading-operations/pages/TradingDashboardPage.tsx` (Erweiterung)
- `its-odin-ui/src/shared/types/sseEvents.ts` (neue/erweiterte Event-Typen — Discriminated Unions)

**TypeScript:** Discriminated Unions fuer SSE-Event-Typen, strict mode. Kein `any`.

## Konzept-Referenzen

- `docs/concept/10-observability.md` — Abschnitt 12 "Dashboard-Komponenten"
- `docs/concept/10-observability.md` — Abschnitt 8 "Alert-Routing und Eskalation"

## Guardrail-Referenzen

- `docs/frontend/guardrails/frontend.md` — "SSE Hooks, Named Events, addEventListener"
- `docs/frontend/guardrails/frontend.md` — "Feature-basierte DDD-Struktur (keine Feature→Feature-Imports)"
- `T:\codebase\its_odin\CLAUDE.md` — Tech-Stack, SSE-Konventionen
- MEMORY.md — "SSE Connection Bugs" (ALLE geloesten Bugs beachten!)

---

## Definition of Done

### 2.1 Code-Qualitaet (Frontend)

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Frontend baut fehlerfrei (`npm run build`)
- [ ] TypeScript strict mode — kein `any`, keine untyped Imports
- [ ] Discriminated Unions fuer alle SSE-Event-Typen in `sseEvents.ts`
- [ ] Keine Magic Strings — Konstanten fuer Event-Namen, Farb-Codes, Timeout-Werte
- [ ] Union-Types statt `enum`-Keyword fuer TypeScript-Enums
- [ ] Keine Feature→Feature-Imports (DDD-Grenze einhalten)
- [ ] CSS Modules fuer Styling
- [ ] Dark Theme konsistent durchgehalten
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + Kommentare)

**Referenz:** `docs/frontend/guardrails/frontend.md`, CLAUDE.md

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Vitest-Unit-Tests fuer Event-Parsing: Alle 8 SSE-Event-Typen werden korrekt geparst und typisiert
- [ ] Vitest-Unit-Tests fuer State-Updates: Dispatch eines `PIPELINE_STATE`-Events aktualisiert korrekt den UI-State
- [ ] Vitest-Unit-Tests fuer Alert-Feed: Neue Alerts werden korrekt nach oben sortiert, Level-Farben korrekt zugeordnet
- [ ] Vitest-Unit-Tests fuer Reconnect-Logik: Backoff-Berechnung (Initial 10s, exponentiell, Max 60s)
- [ ] Testklassen-Namenskonvention: `*.test.ts` / `*.spec.ts`
- [ ] Keine echten HTTP-Verbindungen in Unit-Tests (MSW oder fetch-Mocks)
- [ ] Neue Logik → Unit-Test PFLICHT

**Referenz:** `docs/frontend/guardrails/frontend.md` → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `TradingDashboardPage` mit simuliertem SSE-Stream (MSW oder Test-Double) — Event empfangen, State aktuell, UI korrekt gerendert
- [ ] Integrationstest: Kill-Switch-Flow — Button klicken, ConfirmDialog erscheint, Bestaetigen loest POST aus, visuelles Feedback erscheint
- [ ] Integrationstest: ConnectionBanner zeigt korrekten Status bei Disconnect/Reconnect
- [ ] Testklassen-Namenskonvention: `*.integration.test.ts` oder `*.spec.ts` mit Vitest
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass SSE-Hooks, State-Management und UI-Komponenten korrekt zusammenarbeiten.

### 2.4 Tests — Datenbank

Nicht zutreffend. Frontend-Story ohne Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `usePipelineStream.ts`, `useSseGlobalStream.ts`, `sseEvents.ts`, `TradingDashboardPage.tsx` (relevante Ausschnitte), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei sehr schnellen Event-Bursts (100 Events/s)? Was wenn `instrumentId` im Event nicht mit dem aktuell angezeigten Instrument uebereinstimmt? Wie verhaelt sich React StrictMode mit SSE-Hooks?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Memory-Leaks bei SSE-Hook-Cleanup (useEffect cleanup), Race Conditions bei schnellen State-Updates, korrekte TypeScript-Typisierung der Discriminated Unions, unbehandelte Error-States"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/10-observability.md` Abschnitte 8 und 12 + `docs/frontend/guardrails/frontend.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Dashboard dem Konzept entspricht. Sind alle Dashboard-Komponenten aus Abschnitt 12 implementiert? Stimmen Alert-Level-Farben und Verbindungsstatus-Anzeige mit den Guardrails ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Welche UX-Probleme entstehen in Trading-Dashboards unter Echtzeit-Last? Z.B. flackernde UI bei zu vielen Updates, fehlende Loading-States, fehlendes Error-Boundary-Handling?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. State-Management-Ansatz, Event-Throttling falls implementiert)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- ALLE SSE-Bugs aus MEMORY.md beachten:
  - Heartbeat: `heartbeat-interval-ms=5000`
  - Named SSE Events: `addEventListener("pipelineState", ...)` NICHT `onmessage`
  - React StrictMode: Deferred disconnect (100ms grace period) in SSE-Hooks
  - ConnectionBanner: `hasEverConnectedRef` Guard gegen false-positive red banner
  - ConnectionStatus: `'connecting'` State initial (nicht `'disconnected'`)
  - Backoff: Initial 10s, Max 60s
- Discriminated Unions fuer TypeScript Event-Typen in `sseEvents.ts` — kein `any`-Cast
- Manuelle Verifizierung mit laufendem Backend erforderlich (vor Abschluss der Story)
- Chart-Farben (User-Vorgabe aus MEMORY.md): Bullish `#0EB35B`, Bearish `#FC3243` — fuer P&L-Faerbung verwenden
