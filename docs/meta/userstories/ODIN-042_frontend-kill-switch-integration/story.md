# ODIN-042: Frontend Kill-Switch und Controls Integration

**Modul:** odin-frontend (its-odin-ui)
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** ODIN-038 (Control Endpoints)

---

## Kontext

KillSwitchButton und ConfirmDialog existieren im Frontend, aber die Integration mit den erweiterten Control-Endpoints (Pause, Resume, Safe-Mode) fehlt. Alle Control-Aktionen muessen im Frontend verfuegbar sein.

## Scope

**In Scope:**
- Kill-Switch: Bestehenden Button mit neuem Endpoint verbinden
- Pause/Resume: Per-Pipeline Controls in Pipeline-Detail-View
- Safe-Mode: System-weiter Safe-Mode-Toggle
- Status-Anzeige: System-Mode und Degradation-Mode visuell

**Out of Scope:**
- Backend-Implementierung der Control-Endpoints (ist ODIN-038)
- Konfiguration von Pipeline-Parametern ueber UI
- Mobile-Responsiveness (Desktop-only)

## Akzeptanzkriterien

- [ ] Kill-Switch Button: POST `/api/v1/control/kill`, ConfirmDialog, visuelles Feedback
- [ ] Pause Button pro Pipeline: POST `/api/v1/control/pause/{id}`, Button wechselt zu Resume
- [ ] Resume Button: POST `/api/v1/control/resume/{id}`
- [ ] Safe-Mode Toggle: POST `/api/v1/control/safe-mode`
- [ ] Status-Badge: NORMAL (gruen), QUANT_ONLY (gelb), DATA_HALT (orange), EMERGENCY (rot)
- [ ] Disabled-State bei SSE-Disconnect (Controls nicht nutzbar ohne Verbindung)
- [ ] ConfirmDialog erscheint IMMER vor destruktiven Aktionen (Kill-Switch, Safe-Mode)
- [ ] Ladezustand waehrend REST-Call sichtbar (Button disabled + Spinner)
- [ ] Fehlerfall (HTTP 4xx/5xx) wird dem User angezeigt

## Technische Details

**Dateien:**
- `its-odin-ui/src/domains/trading-operations/components/ControlsPanel.tsx` (Erweiterung)
- `its-odin-ui/src/domains/trading-operations/hooks/useControls.ts` (Erweiterung)
- `its-odin-ui/src/shared/components/KillSwitchButton.tsx` (Integration)

**Patterns:**
- REST POST fuer alle Control-Aktionen (kein WebSocket, kein SSE)
- TypeScript strict: Union-Types fuer Status-Modi
- CSS Modules fuer alle Styles, Dark Theme

## Konzept-Referenzen

- `docs/concept/10-observability.md` -- Abschnitt 3 "Control Interface"
- `docs/concept/06-risk-management.md` -- Abschnitt 4.1 "Schicht 4: Manuell"

## Guardrail-Referenzen

- `docs/frontend/guardrails/frontend.md` -- "REST POST fuer Controls"
- `docs/frontend/guardrails/frontend.md` -- "TypeScript strict, Union-Types"
- `T:/codebase/its_odin/CLAUDE.md` -- Frontend-Architektur (SSE Monitoring + REST POST Controls)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Frontend baut fehlerfrei (`npm run build` bzw. Vite)
- [ ] TypeScript strict -- kein `any`, keine ungetypten Props
- [ ] Union-Types statt `enum`-Keyword fuer Status-Modi (NORMAL, QUANT_ONLY, DATA_HALT, EMERGENCY)
- [ ] CSS Modules fuer alle Styles (kein Inline-Style, kein globales CSS)
- [ ] Dark Theme konsistent eingehalten
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + Kommentare)
- [ ] Feature→Feature-Imports verboten (nur via shared/)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Vitest-Unit-Tests fuer `useControls`-Hook (alle Control-Aktionen)
- [ ] Unit-Tests fuer ConfirmDialog-Logik (erscheint/erscheint nicht je nach Aktion)
- [ ] Unit-Tests fuer Disabled-State-Logik (SSE-Disconnect → Controls disabled)
- [ ] Unit-Tests fuer Status-Badge-Mapping (Status-String → Farbe)
- [ ] Neue Logik → Unit-Test PFLICHT

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Komponenten zusammengeschaltet (nicht alles weggemockt)
- [ ] `ControlsPanel` Integration: Button-Klick → ConfirmDialog → REST POST → Status-Update
- [ ] Fehlerfall-Test: REST-Call schlaegt fehl → Fehlermeldung sichtbar
- [ ] Mindestens 1 Integrationstest der die Kill-Switch-Sequenz End-to-End abdeckt (Button → Confirm → POST)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- reine Frontend-Story, kein Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Komponenten-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Doppelklick auf Kill-Switch, SSE-Disconnect waehrend REST-Call, Safe-Mode-Toggle waehrend bereits im Safe-Mode)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions bei gleichzeitigen Control-Aufrufen, fehlende Ladezustand-Guards (Doppelklick), Fehlerbehandlung bei REST-Calls, Null-Safety"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/10-observability.md` Abschnitt 3 + `docs/concept/06-risk-management.md` Abschnitt 4.1 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht -- insb. korrekte Endpoints, Status-Modi, ConfirmDialog-Pflicht vor destruktiven Aktionen"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Timeout bei fehlender Backend-Antwort, Optimistic UI Update vs. Server-Bestaetigung, gleichzeitiger Kill-Switch von mehreren Browser-Tabs?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State nach initialer Implementierung dokumentiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Optimistic vs. pessimistic UI bei Control-Aktionen)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt nach Test-Sparring
- [ ] Gemini-Review-Abschnitt ausgefuellt nach Review
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- ConfirmDialog muss IMMER vor destruktiven Aktionen stehen (Kill-Switch, Safe-Mode)
- Pause/Resume ist pro Pipeline, nicht global
- Controls muessen bei SSE-Disconnect disabled sein -- User soll keine Befehle schicken koennen, wenn er nicht weiss was passiert
- REST POST fuer Controls (kein WebSocket) -- so ist die Architektur definiert (CLAUDE.md)
