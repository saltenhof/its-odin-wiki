# ODIN-038: Control Endpoints Enhancement

**Modul:** odin-app
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-026 (Degradation Manager)
**Geschaetzter Umfang:** M

---

## Kontext

Das Konzept definiert 4 Control-Aktionen: Kill, Pause/Resume, Safe-Mode. Der aktuelle `ControlController` hat grundlegende Kill-Switch-Endpoints. Pause/Resume und Safe-Mode fehlen. Alle Control-Aktionen muessen idempotent sein und im Audit-Log dokumentiert werden — fuer Nachvollziehbarkeit und regulatorische Anforderungen.

## Scope

**In Scope:**
- Kill-Switch: `POST /api/v1/control/kill` (besteht, verbessern + idempotent machen)
- Pause: `POST /api/v1/control/pause/{instrumentId}` (Pipeline → HALTED)
- Resume: `POST /api/v1/control/resume/{instrumentId}` (HALTED → vorheriger State)
- Safe-Mode: `POST /api/v1/control/safe-mode` (alle Pipelines nur Dashboard, kein Trading)
- Status: `GET /api/v1/control/status` (aktueller System-Status + Degradation)

**Out of Scope:**
- Authentifizierung/Autorisierung der Control-Endpoints (V1: internes Netz, kein Auth)
- Geplante Pauses (Scheduled Pause/Resume)
- Remote-Control ueber externe Systeme

## Akzeptanzkriterien

- [ ] Kill-Switch: Idempotent, EMERGENCY Alert, alle offenen Positionen geschlossen
- [ ] Pause: Einzelne Pipeline anhalten (`HALTED`-State), Position bleibt offen (Broker-Stops schuetzen)
- [ ] Resume: Pipeline fortsetzen im vorherigen State (State wird bei Pause gespeichert)
- [ ] Safe-Mode: Alle Pipelines pausiert, Dashboard funktioniert, kein neues Trading
- [ ] Status: JSON mit `systemMode`, `degradationMode`, `pipelineStates[]`, `activeAlerts`
- [ ] Alle Aktionen: Audit-Log-Eintrag mit Operator-Info und Timestamp
- [ ] Alle Aktionen: `ControlResponse` Record mit Timestamp und Ergebnis (SUCCESS/FAILED + Grund)

## Technische Details

**Dateien:**
- `odin-app/src/main/java/de/its/odin/app/controller/ControlController.java` (Erweiterung)
- `odin-app/src/main/java/de/its/odin/app/dto/ControlStatusResponse.java` (neues Record)
- `odin-app/src/main/java/de/its/odin/app/dto/ControlResponse.java` (neues Record, falls noch nicht vorhanden)

**Abhaengigkeit odin-core:** Pause/Resume erfordert Zugriff auf `PipelineStateMachine` (previousState bei HALTED speichern).

## Konzept-Referenzen

- `docs/concept/10-observability.md` — Abschnitt 3 "Control Interface"
- `docs/concept/10-observability.md` — Abschnitt 3, Tabelle "4 Control-Aktionen"
- `docs/concept/06-risk-management.md` — Abschnitt 4.1 "Schicht 4: Manuell"

## Guardrail-Referenzen

- `T:\codebase\its_odin\CLAUDE.md` — "REST POST (Controls)"
- `docs/frontend/guardrails/frontend.md` — "REST POST fuer Controls"
- `docs/backend/guardrails/module-structure.md` — Abhaengigkeitsregeln

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-app`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (`ControlStatusResponse`, `ControlResponse`)
- [ ] ENUM statt String fuer endliche Mengen (z.B. `ControlResult.SUCCESS / FAILED`)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.app.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer Kill-Switch: Idempotenz (zweiter Aufruf aendert nichts), EMERGENCY-Alert wird ausgeloest
- [ ] Unit-Tests fuer Pause: Pipeline geht in HALTED, vorheriger State wird gespeichert
- [ ] Unit-Tests fuer Resume: Pipeline geht zurueck in gespeicherten State
- [ ] Unit-Tests fuer Safe-Mode: Alle Pipelines werden pausiert
- [ ] Unit-Tests fuer Status-Endpoint: Korrekte Aggregation von systemMode, degradationMode, pipelineStates
- [ ] Unit-Tests fuer Idempotenz: Alle Control-Aktionen koennen mehrfach aufgerufen werden ohne Fehler
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces und PipelineStateMachine
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: Pause-Endpoint → Pipeline-State wechselt zu HALTED, State gespeichert → Resume-Endpoint → Pipeline zurueck im vorherigen State (reale PipelineStateMachine eingebunden)
- [ ] Integrationstest: Kill-Switch → EMERGENCY-Alert im Audit-Log nachweisbar
- [ ] Integrationstest: Status-Endpoint liefert korrektes JSON mit allen Feldern
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass ControlController, PipelineStateMachine und EventLog korrekt zusammenarbeiten.

### 2.4 Tests — Datenbank

Nicht zutreffend. Control-Endpoints operieren auf dem In-Memory-Zustand von Pipelines und leiten Audit-Log-Eintraege weiter. Direkte DB-Tests sind in dieser Story nicht erforderlich (Audit-Log-Persistenz wird durch ODIN-031 abgedeckt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `ControlController.java` (Erweiterung), `ControlStatusResponse.java`, `ControlResponse.java`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei Resume wenn keine Pipeline pausiert ist? Was wenn Kill-Switch ausgeloest wird waehrend eine Order in Bearbeitung ist? Wie verhaelt sich Safe-Mode bei gleichzeitiger Aktivierung durch zwei Operatoren?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions bei gleichzeitigen Control-Aktionen, Idempotenz-Implementierung, korrekte State-Speicherung bei Pause/Resume, Null-Safety"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/10-observability.md` Abschnitt 3 + `docs/concept/06-risk-management.md` Abschnitt 4.1 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die 4 Control-Aktionen dem Konzept entsprechen. Ist Idempotenz korrekt implementiert? Sind alle Audit-Log-Eintraege vorhanden? Stimmt der Status-Response mit dem definierten Schema ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Welche praktischen Probleme entstehen bei manuellen Control-Aktionen in Echtzeit-Trading-Systemen? Was fehlt (z.B. Operator-Identitaet, Rate-Limiting, Confirmation-Token)?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. State-Speicherung bei Pause: In-Memory oder DB?)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- Pause/Resume erfordert State-Speicherung im `PipelineStateMachine` (vorheriger State vor HALTED speichern)
- `PipelineStateMachine` hat bereits `HALTED` State — aber die Resume-Logik (zurueck zum vorherigen State) fehlt noch
- ODIN-026 (Degradation Manager) muss abgeschlossen sein, damit Safe-Mode den `DegradationMode` korrekt setzen kann
