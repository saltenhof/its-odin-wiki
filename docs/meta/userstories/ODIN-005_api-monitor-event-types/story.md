# ODIN-005: 1-Minute Monitor Event Types im API-Modul

**Modul:** odin-api
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** keine

---

## Context

Das Konzept definiert 9 Monitor-Event-Typen die auf 1-Minute-Bars erkannt werden (zwischen den 3m-Decision-Bars). Diese Events triggern LLM-Calls, Alerts, oder Kill-Switch-Eskalation. Die API braucht die entsprechenden Event-Typen und ein Event-Record.

## Scope

**In Scope:**
- Neues Enum `MonitorEventType` mit 9 Werten gemaess Konzept 01, Abschnitt 8
- Neues Record `MonitorEvent` in `de.its.odin.api.event`
- Enum `EscalationAction` (LOG_ONLY, TRIGGER_LLM_CALL, ALERT, KILL_SWITCH)
- Default-EscalationAction pro MonitorEventType (per Konzept Tabelle)

**Out of Scope:**
- Detektor-Implementierung in odin-data (kommt in ODIN-007)
- Listener-Port in odin-api -- dieser Scope gehört zu ODIN-007 oder ist in den Notizen bereits als noetig erkannt
- Persistierung von MonitorEvents

## Acceptance Criteria

- [ ] Enum `MonitorEventType`: VOLUME_SPIKE, PRICE_BREAK_LEVEL, MOMENTUM_DIVERGENCE, SPREAD_EXPANSION, STALE_QUOTE_WARNING, UNUSUAL_ACTIVITY, RANGE_EXPANSION, VWAP_CROSS, EXHAUSTION_SIGNAL
- [ ] Record `MonitorEvent(MonitorEventType type, String instrumentId, Instant marketTime, double value, double threshold, EscalationAction escalation, String runId)`
- [ ] Enum `EscalationAction` mit 4 Werten: LOG_ONLY, TRIGGER_LLM_CALL, ALERT, KILL_SWITCH
- [ ] Jeder MonitorEventType hat einen Default-EscalationAction (per Konzept Tabelle, kodiert im Enum)

## Technical Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/model/MonitorEventType.java`
- `odin-api/src/main/java/de/its/odin/api/model/EscalationAction.java`
- `odin-api/src/main/java/de/its/odin/api/event/MonitorEvent.java`

## Concept References

- `docs/concept/01-data-pipeline.md` -- Abschnitt 8 "1-Minute Monitor Events"
- `docs/concept/01-data-pipeline.md` -- Abschnitt 8, Tabelle "9 Event-Typen mit Eskalationsregeln"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "ENUM statt String fuer endliche Mengen"
- `docs/backend/guardrails/module-structure.md` -- "odin-api: Records fuer DTOs"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- "Events sind immutable"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (`MonitorEvent` als Record -- immutable)
- [ ] ENUM statt String fuer endliche Mengen (`MonitorEventType`, `EscalationAction` als Enums)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen -- JavaDoc pro Enum-Wert mit Beschreibung des ausloesenden Ereignisses und Default-Eskalation
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Keine Abhaengigkeit ausser JDK + Validation-API (odin-api Regel)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Default-EscalationAction fuer alle 9 MonitorEventType-Werte korrekt
- [ ] Unit-Test: `MonitorEvent` korrekt konstruiert mit allen Pflichtfeldern
- [ ] Unit-Test: Enum-Vollstaendigkeit fuer `EscalationAction` (alle 4 Werte)
- [ ] Testklassen-Namenskonvention: `MonitorEventTypeTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `MonitorEvent` mit `MonitorEventType.KILL_SWITCH`-Eskalation korrekt gebaut
- [ ] Integrationstest: Alle 9 MonitorEventType-Werte werden zu gueltigen `MonitorEvent`-Instanzen
- [ ] Testklassen-Namenskonvention: `MonitorEventIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- odin-api enthaelt keine Datenbankzugriffe.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `MonitorEventType.java`, `EscalationAction.java`, `MonitorEvent.java`, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. `value` = `threshold` exakt, `instrumentId` leer oder null, `marketTime` aus der Zukunft, welcher EscalationAction-Wert fehlt moeglicherweise)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Null-Safety bei `instrumentId`/`runId`, Immutabilitaet des Records, Enum-Vollstaendigkeit"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/01-data-pipeline.md` Abschnitt 8 (inkl. Tabelle) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 9 Event-Typen korrekt abgebildet sind und die Default-Eskalationsregeln der Konzept-Tabelle entsprechen"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Erweiterbarkeit um neue Event-Typen, ob `runId` nullable sein sollte, Serialisierungs-Anforderungen fuer SSE-Transport"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. ob Default-EscalationAction im Enum oder in einer separaten Lookup-Tabelle kodiert wird)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Die 9 Event-Typen und ihre Eskalationsregeln stehen exakt in Konzept 01, Abschnitt 8
- MonitorEvents werden von odin-data emittiert und von odin-core konsumiert (analog DataQualityEvent)
- Neuer Listener-Port in odin-api noetig: `MonitorEventListener.onMonitorEvent(MonitorEvent)` -- ob dieser Port hier oder in ODIN-007 erstellt wird, ist eine Implementierungsentscheidung; im `protocol.md` dokumentieren
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
