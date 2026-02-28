# ODIN-006: Cycle-Tracking-Modell im API-Modul

**Modul:** odin-api
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** keine

---

## Context

Das Konzept unterscheidet Trade, Cycle, und Tranche als hierarchische Begriffe. Ein "Cycle" ist ein Neueinstieg nach Flat-Position (Re-Entry). Die aktuelle Codebase trackt nur einzelne Trades, nicht Zyklen. Fuer Multi-Cycle-Day braucht es ein explizites Cycle-Modell.

## Scope

**In Scope:**
- Neues Record `CycleContext` mit cycleNumber, maxCycles, cycleStartTime, previousCycleResult
- Erweiterung von `RunContext` oder `AccountRiskState` um cycleCount und maxCyclesReached
- Neues Enum `CycleEntryReason` (INITIAL, RE_ENTRY_AFTER_TARGET, RE_ENTRY_AFTER_STOP, RE_ENTRY_LLM_APPROVED)

**Out of Scope:**
- Cycle-Management-Logik in odin-core oder odin-execution (nur API-Modell hier)
- Sizing-Faktor-Berechnung fuer Cycle 2+ (kommt in ODIN-022/025)
- Persistierung von CycleContext

## Acceptance Criteria

- [ ] Record `CycleContext(int cycleNumber, int maxCyclesPerInstrument, int maxCyclesGlobal, Instant cycleStartTime, Double previousCyclePnl, CycleEntryReason entryReason)`
- [ ] Enum `CycleEntryReason` mit 4 Werten: INITIAL, RE_ENTRY_AFTER_TARGET, RE_ENTRY_AFTER_STOP, RE_ENTRY_LLM_APPROVED
- [ ] `AccountRiskState` erhaelt neues Feld `int totalCyclesCompleted`
- [ ] Validierung: cycleNumber >= 1, maxCyclesPerInstrument >= 1, maxCyclesGlobal >= 1

## Technical Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/dto/CycleContext.java`
- `odin-api/src/main/java/de/its/odin/api/model/CycleEntryReason.java`

## Concept References

- `docs/concept/00-overview.md` -- Abschnitt 5 "Normative Terminologie: Trade, Cycle, Tranche"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 7 "Multi-Cycle-Day"
- `docs/concept/06-risk-management.md` -- Abschnitt 1.2 "Tages-Ebene: Max. Zyklen"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "odin-api: Records fuer DTOs"
- `docs/backend/guardrails/module-structure.md` -- "ENUM statt String fuer endliche Mengen"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (`CycleContext` als Record)
- [ ] ENUM statt String fuer endliche Mengen (`CycleEntryReason` als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen -- insbesondere JavaDoc pro Enum-Wert mit Erlaeuterung des Eintritts-Grundes
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Keine Abhaengigkeit ausser JDK + Validation-API (odin-api Regel)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: `CycleContext` mit `cycleNumber=1` (erster Zyklus) korrekt konstruiert
- [ ] Unit-Test: `CycleContext` mit `previousCyclePnl=null` (erster Zyklus -- kein vorangegangener P&L)
- [ ] Unit-Test: Validierung schlaegt an bei `cycleNumber=0`
- [ ] Unit-Test: Alle 4 `CycleEntryReason`-Werte korrekt als Enum vorhanden
- [ ] Testklassen-Namenskonvention: `CycleContextTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `CycleContext` Cycle 2 -- `previousCyclePnl` gesetzt, `entryReason=RE_ENTRY_AFTER_TARGET`
- [ ] Integrationstest: `AccountRiskState` mit erweitertem `totalCyclesCompleted`-Feld korrekt befuellt
- [ ] Testklassen-Namenskonvention: `CycleTrackingIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- odin-api enthaelt keine Datenbankzugriffe.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `CycleContext.java`, `CycleEntryReason.java`, `AccountRiskState`-Erweiterung, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. cycleNumber > maxCyclesGlobal, negativer previousCyclePnl, was passiert wenn maxCyclesPerInstrument > maxCyclesGlobal)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Null-Safety bei `previousCyclePnl`, Konsistenz zwischen `cycleNumber` und `maxCyclesPerInstrument`, Validierungslogik"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/00-overview.md` Abschnitt 5 + `docs/concept/03-strategy-logic.md` Abschnitt 7 + `docs/concept/06-risk-management.md` Abschnitt 1.2 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob das Cycle-Modell die normative Terminologie (Trade vs. Cycle vs. Tranche) korrekt abbildet und ob alle Limits aus dem Konzept im Modell representiert sind"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. wie Cycle-Grenzen bei parallelen Pipelines durchgesetzt werden, Thread-Safety von `totalCyclesCompleted`, Verhalten bei Tages-Reset"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. ob `maxCyclesPerInstrument` und `maxCyclesGlobal` im Record oder in Konfiguration gehalten werden)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Global max 5 Cycles/Tag, per Pipeline max 3 (konfigurierbar). Globales Limit dominiert.
- CycleContext wird pro Pipeline gefuehrt und bei jedem Re-Entry aktualisiert
- Cycle 2+ hat reduzierten Sizing-Faktor (Default 0.6-0.8) -- das wird in ODIN-022/025 genutzt
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
