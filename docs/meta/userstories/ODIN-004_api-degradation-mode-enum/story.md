# ODIN-004: Degradation-Mode-Enum und System-Health-Model

**Modul:** odin-api
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** keine

---

## Context

Das Konzept definiert drei Degradation Modes (QUANT_ONLY, DATA_HALT, EMERGENCY) als Zwischenstufen vor dem Kill-Switch. Die aktuelle Codebase hat nur binaere Zustaende (normal oder DAY_STOPPED). Degradation Modes erlauben kontrolliertes Herunterfahren statt abruptem Halt.

## Scope

**In Scope:**
- Neues Enum `DegradationMode` in `de.its.odin.api.model`: NORMAL, QUANT_ONLY, DATA_HALT, EMERGENCY
- Neues Record `SystemHealthState` das den aktuellen Degradation Mode, aktive Alerts, und Pipeline-Health aggregiert
- Erweiterung von `PipelineState` um DEGRADED-State oder Kombination PipelineState + DegradationMode

**Out of Scope:**
- Implementierung des Degradation-Mechanismus in odin-core oder odin-data (nur API-Modell hier)
- Kill-Switch-Integration -- kommt in anderen Stories
- Persistierung von SystemHealthState

## Acceptance Criteria

- [ ] Enum `DegradationMode` mit 4 Werten und JavaDoc pro Wert (Verhalten laut Konzept)
- [ ] Record `SystemHealthState(DegradationMode mode, List<String> activeAlerts, Instant since)`
- [ ] `DegradationMode` hat Methode `allowsNewEntries()` (NORMAL=true, QUANT_ONLY=false, DATA_HALT=false, EMERGENCY=false)
- [ ] `DegradationMode` hat Methode `allowsExits()` (NORMAL=true, QUANT_ONLY=true, DATA_HALT=risk-reducing only, EMERGENCY=true)

## Technical Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/model/DegradationMode.java`
- `odin-api/src/main/java/de/its/odin/api/dto/SystemHealthState.java`

## Concept References

- `docs/concept/06-risk-management.md` -- Abschnitt 6 "Degradation Modes (Normativ)"
- `docs/concept/11-edge-cases.md` -- Abschnitt 5 "Degradation Modes und Kaskade"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "ENUM statt String fuer endliche Mengen"
- `docs/backend/guardrails/module-structure.md` -- "odin-api: Records fuer DTOs"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (`SystemHealthState` als Record)
- [ ] ENUM statt String fuer endliche Mengen (`DegradationMode` als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen -- insbesondere JavaDoc auf jedem Enum-Wert mit Verhaltens-Beschreibung
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Keine Abhaengigkeit ausser JDK + Validation-API (odin-api Regel)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: `allowsNewEntries()` fuer alle 4 DegradationMode-Werte
- [ ] Unit-Test: `allowsExits()` fuer alle 4 DegradationMode-Werte
- [ ] Unit-Test: `SystemHealthState` korrekt konstruiert mit nicht-leerem `activeAlerts`
- [ ] Unit-Test: `SystemHealthState` korrekt konstruiert mit leerer `activeAlerts`-Liste
- [ ] Testklassen-Namenskonvention: `DegradationModeTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendiger `SystemHealthState` mit `DegradationMode.QUANT_ONLY` -- `allowsNewEntries()=false`, `allowsExits()=true`
- [ ] Integrationstest: Zustandsuebergang von NORMAL zu DATA_HALT -- korrekte Methodenrueckgaben
- [ ] Testklassen-Namenskonvention: `DegradationModeIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- odin-api enthaelt keine Datenbankzugriffe.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `DegradationMode.java`, `SystemHealthState.java`, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. DATA_HALT mit `allowsExits()=true` vs. "risk-reducing only" Semantik, EMERGENCY-Sonderbehandlung, `since`-Zeitstempel in der Vergangenheit)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety bei `activeAlerts`, semantische Konsistenz von `allowsExits()` bei DATA_HALT"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/06-risk-management.md` Abschnitt 6 + `docs/concept/11-edge-cases.md` Abschnitt 5 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob `allowsNewEntries()` und `allowsExits()` fuer alle 4 Modes dem Konzept entsprechen, insbesondere DATA_HALT (risk-reducing only)"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Uebergangsbedingungen zwischen Modes, ob `DegradationMode` orthogonal zu `PipelineState` ist, Monitoring-Implikationen"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. warum `DegradationMode` NICHT als PipelineState-Erweiterung)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- QUANT_ONLY: LLM Timeout/invalid, keine neuen Entries, bestehende Positionen managen (Trailing, Targets), EOD close
- DATA_HALT: Daten stale, keine neuen Orders, nur risk-reducing Exits, bei Persistenz Kill-Switch
- EMERGENCY: Kill-Switch, manuelle Eskalation, Retry alle 30s
- DegradationMode NICHT als PipelineState-Erweiterung -- es ist eine orthogonale Dimension (Pipeline kann POSITIONED + QUANT_ONLY sein)
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
