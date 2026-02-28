# ODIN-026: Degradation Mode State Machine

**Modul:** odin-core
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-004 (DegradationMode Enum)

---

## Context

Das Konzept fordert kontrollierte Degradation statt sofortigem Kill-Switch. Bei LLM-Ausfall -> QUANT_ONLY, bei Daten-Problemen -> DATA_HALT, bei persistentem Ausfall -> EMERGENCY -> Kill-Switch. Die aktuelle Codebase hat nur binaeres Kill-Switch.

## Scope

**In Scope:**
- Neue Klasse `DegradationManager` in odin-core
- State Machine: NORMAL -> QUANT_ONLY -> DATA_HALT -> EMERGENCY -> Kill-Switch
- Trigger-Mapping: LLM-Failure -> QUANT_ONLY, Data-Feed-Stale -> DATA_HALT, Broker-Reject -> EMERGENCY
- Integration mit TradingPipeline: Degradation-Mode beeinflusst erlaubte Aktionen
- Integration mit KillSwitchService: EMERGENCY eskaliert automatisch zu Kill-Switch nach Timeout

**Out of Scope:**
- Aenderungen am Kill-Switch-Mechanismus selbst (das ist ODIN-028)
- Frontend-Anzeige des Degradation-Modes (separates Frontend-Ticket)
- Automatische Wiederherstellung von DATA_HALT oder EMERGENCY (nur QUANT_ONLY -> NORMAL)

## Acceptance Criteria

- [ ] `DegradationManager` verwaltet aktuellen DegradationMode
- [ ] LLM 3 Failures in Folge -> QUANT_ONLY (kein sofortiger Halt)
- [ ] Data-Feed stale > 30s -> DATA_HALT (nur risk-reducing Exits)
- [ ] Data-Feed stale > 60s -> EMERGENCY -> Kill-Switch
- [ ] Broker-Reject / Cannot-Liquidate -> EMERGENCY sofort
- [ ] QUANT_ONLY: Keine neuen Entries, bestehende Positionen managen, EOD close
- [ ] DATA_HALT: Keine neuen Orders, nur risk-reducing Exits
- [ ] Degradation ist reversibel: Wenn LLM wieder verfuegbar -> NORMAL
- [ ] EventLog: DEGRADATION_CHANGED mit oldMode, newMode, reason
- [ ] Alert-Emission bei jedem Mode-Wechsel

## Technical Details

**Dateien:**
- `odin-core/src/main/java/de/its/odin/core/service/DegradationManager.java` (neue Klasse)
- `odin-core/src/main/java/de/its/odin/core/pipeline/TradingPipeline.java` (Integration)
- `odin-core/src/main/java/de/its/odin/core/service/KillSwitchService.java` (EMERGENCY-Eskalation)

**Konfiguration:**
- `odin.core.degradation.llm-failure-threshold` (Default: 3)
- `odin.core.degradation.data-stale-halt-seconds` (Default: 30)
- `odin.core.degradation.data-stale-emergency-seconds` (Default: 60)

**Patterns:**
- `DegradationManager` als Singleton (eine Instanz fuer alle Pipelines)
- TradingPipeline prueft DegradationMode VOR der Decision Loop
- Zustandsuebergaenge sind strikt unidirektional (ausser QUANT_ONLY -> NORMAL)

## Concept References

- `docs/concept/06-risk-management.md` -- Abschnitt 6 "Degradation Modes (Normativ)"
- `docs/concept/06-risk-management.md` -- Abschnitt 6, Tabelle "3 Degradation Modes"
- `docs/concept/11-edge-cases.md` -- Abschnitt 5 "Degradation Modes und Kaskade"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Singleton-Instanziierung
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `T:\codebase\its_odin\CLAUDE.md` -- Instanziierungsmodell (Singletons: GlobalRiskManager)

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. Timeout-Schwellen)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (`DegradationMode` als ENUM)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.core.degradation.*` fuer Konfiguration
- [ ] Port-Abstraktion: `EventLog`-Interface aus `de.its.odin.api.port` fuer Logging

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: Alle Transitionen (NORMAL -> QUANT_ONLY -> DATA_HALT -> EMERGENCY)
- [ ] Unit-Tests: Recovery-Transition (QUANT_ONLY -> NORMAL wenn LLM wieder verfuegbar)
- [ ] Unit-Tests: EMERGENCY -> Kill-Switch Eskalation
- [ ] Unit-Tests: Broker-Reject loest sofort EMERGENCY aus (kein gradueller Weg)
- [ ] Unit-Tests: LLM-Failure-Counter (2 Failures = noch NORMAL, 3 = QUANT_ONLY)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `EventLog` und `KillSwitchService`
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `DegradationManager` + `TradingPipeline` — Decision Loop wird bei DATA_HALT blockiert
- [ ] Integrationstest: `DegradationManager` + `KillSwitchService` — EMERGENCY eskaliert korrekt
- [ ] Integrationstest: LLM-Recovery setzt Failure-Counter zurueck und stellt NORMAL wieder her
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — `DegradationManager` ist ein In-Memory-Singleton ohne Datenbankzugriff. Events werden ueber `EventLog`-Port geschrieben (in Tests gemockt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `DegradationManager`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn Data-Feed-Stale und LLM-Failure gleichzeitig auftreten? Was bei Degradation waehrend aktivem EOD-Close?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Thread-Safety beim Singleton-Zustand, korrekte Atomaritaet bei Mode-Wechseln, Null-Safety"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/06-risk-management.md` (Abschnitt 6) + `docs/concept/11-edge-cases.md` (Abschnitt 5) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Trigger-Mapping, erlaubte Aktionen pro Mode und Kaskade dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche Degradations-Szenarien koennten in der Praxis auftreten, die im Konzept nicht behandelt sind? (z.B. intermittierender LLM-Ausfall, partieller Datenfeed-Ausfall)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Thread-Safety-Strategie fuer Singleton)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- "Degradation vor Kill-Switch" (Konzept 06, Abschnitt 6) -- bei einzelnen Quality-Gate-Verletzungen zunaechst degradieren
- DegradationManager ist ein Singleton (eine Instanz fuer alle Pipelines)
- TradingPipeline prueft DegradationMode VOR der Decision Loop
