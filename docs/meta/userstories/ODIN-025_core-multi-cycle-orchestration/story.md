# ODIN-025: Multi-Cycle-Day Orchestration

**Modul:** odin-core
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-006 (Cycle Tracking Model), ODIN-022 (Multi-Cycle OMS), ODIN-017 (LLM Tactical Controller)

---

## Context

Das Konzept definiert Multi-Cycle-Day: Bis zu 3 Zyklen pro Instrument, 5 global. Nach Flat-Position kann die Pipeline unter bestimmten Bedingungen re-entren. Die aktuelle TradingPipeline geht nach Exit zurueck zu OBSERVING aber hat keine Cycle-Zaehlung, keine Guardrails, und keine LLM-Pflicht fuer Re-Entry.

## Scope

**In Scope:**
- Cycle-Counter und Budget-Tracking in TradingPipeline
- Re-Entry Guardrails: Cooling-Off nach Stop-Out, Mindestgewinn fuer Re-Entry, LLM-Pflicht
- Koordination mit GlobalRiskManager (globaler Cycle-Count)
- Cycle-Wechsel-Logik: OMS-Reset, Cycle-Context-Update, EventLog
- FSM-Erweiterung: Optionaler Zustand COOLING_OFF zwischen Cycles

**Out of Scope:**
- OMS-interne Cycle-Sizing-Logik (das ist ODIN-022)
- LLM-Tactical-Controller-Implementierung (das ist ODIN-017)
- Frontend-Darstellung von Cycle-Status

## Acceptance Criteria

- [ ] TradingPipeline trackt aktuelle Cycle-Nummer pro Instrument
- [ ] Max. 3 Cycles pro Instrument/Tag (konfigurierbar)
- [ ] Max. 5 Cycles global/Tag (dominiert ueber per-Instrument-Limit)
- [ ] Re-Entry nach Stop-Out: 15-30 Minuten Cooling-Off (konfigurierbar)
- [ ] Re-Entry nach Target-Exit: LLM muss ENTER empfehlen (Pflicht-Call)
- [ ] Re-Entry nach Profit-Exit: Budget-Gate pruefen (verbleibendes Budget > Minimum)
- [ ] Cycle-Wechsel: OMS.reset() + neuer CycleContext + EventLog CYCLE_START
- [ ] GlobalRiskManager.onCycleComplete() aktualisiert globalen Counter
- [ ] Kein Re-Entry bei Regime UNCERTAIN oder TREND_DOWN (Konzept 03, Abschnitt 7)

## Technical Details

**Dateien:**
- `odin-core/src/main/java/de/its/odin/core/pipeline/TradingPipeline.java` (Cycle-Logik)
- `odin-core/src/main/java/de/its/odin/core/service/GlobalRiskManager.java` (Cycle-Counting)
- `odin-core/src/main/java/de/its/odin/core/pipeline/PipelineStateMachine.java` (optional COOLING_OFF)

**Konfiguration:**
- `odin.core.pipeline.max-cycles-per-instrument` (Default: 3)
- `odin.core.pipeline.max-cycles-global` (Default: 5)
- `odin.core.pipeline.stop-out-cooldown-minutes` (Default: 20)

## Concept References

- `docs/concept/03-strategy-logic.md` -- Abschnitt 7 "Multi-Cycle-Day"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 7, Tabelle "Guardrails"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 7.3 "LLM-Rolle bei Multi-Cycle"
- `docs/concept/06-risk-management.md` -- Abschnitt 1.2 "Max. Zyklen pro Pipeline/Tag"
- `docs/concept/06-risk-management.md` -- Abschnitt 2.2 "Cycle-2+ Sizing"
- `docs/concept/06-risk-management.md` -- Abschnitt 8 "Cooldown nach Stop-Out"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, odin-core Abhaengigkeitsregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `T:\codebase\its_odin\CLAUDE.md` -- Instanziierungsmodell (Singletons vs. Pro-Pipeline)

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (z.B. `CycleContext`)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.core.pipeline.*` fuer Konfiguration
- [ ] Port-Abstraktion: `EventLog`- und `LlmAnalyst`-Interfaces aus `de.its.odin.api.port` verwenden

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: 3 Cycles pro Instrument (4. wird blockiert)
- [ ] Unit-Tests: 5 Cycles global (6. wird blockiert, auch wenn Instrument-Limit nicht erreicht)
- [ ] Unit-Tests: Cooling-Off nach Stop-Out (kein Re-Entry vor Ablauf)
- [ ] Unit-Tests: LLM-Pflicht fuer Re-Entry nach Target-Exit (kein Re-Entry ohne LLM ENTER)
- [ ] Unit-Tests: Budget-Gate (kein Re-Entry wenn Budget erschoepft)
- [ ] Unit-Tests: Regime-Gate (kein Re-Entry in UNCERTAIN/TREND_DOWN)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `EventLog`, `LlmAnalyst`, `GlobalRiskManager`
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendiger 2-Cycle-Ablauf: Entry1 -> Exit1 -> CoolingOff -> Entry2 -> Exit2
- [ ] Integrationstest: Globales Cycle-Limit sperrt alle Pipelines nach Erreichen
- [ ] Integrationstest: `TradingPipeline` + `GlobalRiskManager` zusammengeschaltet — Cycle-State bleibt konsistent
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — Cycle-Logik ist In-Memory in TradingPipeline und GlobalRiskManager. Events werden ueber `EventLog`-Port geschrieben (in Tests gemockt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: TradingPipeline-Erweiterung, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn Cooling-Off durch EOD unterbrochen wird? Was bei gleichzeitigem Kill-Switch-Trigger?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Thread-Safety bei GlobalRiskManager Cycle-Counter, korrekte Atomaritaet bei Cycle-Wechsel"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/03-strategy-logic.md` (Abschnitt 7) + `docs/concept/06-risk-management.md` (Abschnitte 1.2, 2.2, 8) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Cycle-Limits, Re-Entry-Guardrails, Cooling-Off-Timing und Regime-Gate dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche Multi-Cycle-Szenarien koennten in der Praxis auftreten, die im Konzept nicht behandelt sind? (z.B. Crash waehrend Cooling-Off, LLM-Timeout beim Pflicht-Call)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. ob COOLING_OFF als FSM-State oder als Flag realisiert wird)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- Das globale 5-Cycles-Limit dominiert ueber per-Pipeline-Limits (Konzept 06, Abschnitt 1.2)
- "Gewinne erhoehen das Budget NICHT" -- zentrale Design-Entscheidung
- Cycle-Wechsel ist KEIN Neustart der Pipeline. Nur OMS wird resettet, DataPipeline und KPI laufen weiter
- COOLING_OFF State ist optional im FSM -- alternativ kann OBSERVING + Cooling-Flag verwendet werden
