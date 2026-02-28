# ODIN-027: EOD Forced Close Timing Logic

**Modul:** odin-core
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-008 (Session Boundaries)

---

## Context

Das Konzept fordert EOD-Flat: Alle Positionen muessen vor RTH-Close geschlossen sein. Die aktuelle LifecycleManager hat rudimentaere EOD-Logik aber keine gestaffelte Close-Sequenz. Das Konzept definiert: 15 Minuten vor Close -> FORCED_CLOSE State -> Market-Orders fuer alle offenen Positionen.

## Scope

**In Scope:**
- Gestaffelte EOD-Close-Logik in LifecycleManager
- Phase 1 (15 Min vor Close): Pipeline -> FORCED_CLOSE, keine neuen Entries
- Phase 2 (10 Min vor Close): Offene Limit-Exits werden zu Market-Orders konvertiert
- Phase 3 (5 Min vor Close): Emergency-Close aller verbleibenden Positionen
- Phase 4 (RTH Close): Pipeline -> EOD, alle Orders storniert
- Integration mit SessionBoundaries fuer exakte Timing-Berechnung

**Out of Scope:**
- After-Hours-Handel oder Extended-Hours-Close
- Positions-Closing-Logik selbst (liegt im OMS/BrokerGateway)
- Session-Boundary-Berechnung (das ist ODIN-008)

## Acceptance Criteria

- [ ] LifecycleManager prueft periodisch die verbleibende RTH-Zeit
- [ ] T-15min: Alle Pipelines -> FORCED_CLOSE State
- [ ] T-10min: Alle offenen Limit-Exit-Orders werden storniert und als Market-Orders resubmitted
- [ ] T-5min: Emergency-Close fuer alle noch offenen Positionen (Market-Orders)
- [ ] T-0: Pipeline -> EOD, alle verbleibenden Orders storniert
- [ ] Konfigurierbare Timing-Schwellen in CoreProperties
- [ ] EventLog: EOD_PHASE_1, EOD_PHASE_2, EOD_PHASE_3, EOD_COMPLETE
- [ ] Kein neuer Entry nach FORCED_CLOSE (FSM-Guard)

## Technical Details

**Dateien:**
- `odin-core/src/main/java/de/its/odin/core/service/LifecycleManager.java` (Erweiterung)
- `odin-core/src/main/java/de/its/odin/core/config/CoreProperties.java` (Timing-Schwellen)

**Konfiguration:**
- `odin.core.eod.forced-close-minutes-before-close` (Default: 15)
- `odin.core.eod.market-order-convert-minutes-before-close` (Default: 10)
- `odin.core.eod.emergency-close-minutes-before-close` (Default: 5)

**Patterns:**
- ALLE Zeitberechnungen ueber `MarketClock`-Port (Sim-Kompatibilitaet)
- FSM-Guard in `PipelineStateMachine`: Kein Transition nach ENTERING wenn State=FORCED_CLOSE

## Concept References

- `docs/concept/03-strategy-logic.md` -- Abschnitt 1 "Tagesablauf Phase 5: RTH-Close-Countdown"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 2 "Pipeline FSM: FORCED_CLOSE -> EOD"
- `docs/concept/07-oms-execution.md` -- Abschnitt 10 "Forced Close Policy"
- `docs/concept/06-risk-management.md` -- Abschnitt 4.3 "Kill-Switch Aktion"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, odin-core Abhaengigkeitsregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `T:\codebase\its_odin\CLAUDE.md` -- "MarketClock verwenden: kein Instant.now() im Trading-Codepfad"

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (alle Timing-Defaults als Konstanten)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (EOD-Phase als ENUM)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.core.eod.*` fuer Konfiguration
- [ ] Kein `Instant.now()` im Code — ausschliesslich `MarketClock.now()` verwenden

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: Alle 4 Phasen mit `SimClock` (Zeitsteuerung ueber simulierte MarketClock)
- [ ] Unit-Tests: Market-Order-Konvertierung (Limit -> Market) bei T-10
- [ ] Unit-Tests: Emergency-Close bei T-5 fuer verbleibende offene Positionen
- [ ] Unit-Tests: Kein Entry nach FORCED_CLOSE (FSM-Guard)
- [ ] Unit-Tests: EOD-Events werden in korrekter Reihenfolge geloggt
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `MarketClock`, `EventLog`, `BrokerGateway` Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `LifecycleManager` + `PipelineStateMachine` + `SimClock` — komplette EOD-Sequenz durchlaufen
- [ ] Integrationstest: Position offen bei T-15 wird korrekt durch alle Phasen geschlossen
- [ ] Integrationstest: Neuer Entry-Versuch nach FORCED_CLOSE wird vom FSM-Guard abgelehnt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — EOD-Close-Timing-Logik ist In-Memory. Events werden ueber `EventLog`-Port geschrieben (in Tests gemockt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: EOD-Close-Logik in `LifecycleManager`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn T-5 Cancel-Requests vom Broker abgelehnt werden? Was bei Crash exakt in der EOD-Sequenz?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, moegliche Race Conditions bei der Phase-Steuerung, korrekte Fehlerbehandlung wenn Cancel/Market-Order-Submit fehlschlaegt"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/03-strategy-logic.md` (Abschnitt 1, 2) + `docs/concept/07-oms-execution.md` (Abschnitt 10) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Phase-Timing, FSM-Transitionen und Market-Order-Konvertierung dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche EOD-Szenarien koennten in der Praxis auftreten, die im Konzept nicht behandelt sind? (z.B. IB-Verbindungsabbruch waehrend EOD-Sequenz, haengender Market-Order-Fill)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. wie periodische RTH-Zeit-Pruefung implementiert ist)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- ALLE Zeitberechnungen ueber MarketClock! (Sim-Kompatibilitaet)
- NYSE RTH Close = 16:00 ET -> FORCED_CLOSE ab 15:45 ET
- MarketOrderJustification.FORCED_CLOSE ist einer der erlaubten Market-Order-Faelle (Konzept 07)
