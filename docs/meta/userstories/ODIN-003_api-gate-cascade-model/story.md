# ODIN-003: Gate-Cascade-Modell im API-Modul

**Modul:** odin-api
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** keine

---

## Context

Das Konzept (03-strategy-logic.md, Abschnitt 4.2) definiert eine 7-Gate-Kaskade die den bisherigen gewichteten Quant-Score ersetzt. Jedes Gate ist ein harter Pass/Fail-Check. Die API braucht ein Modell fuer das Gate-Resultat, damit Brain und Execution einheitlich damit arbeiten koennen.

## Scope

**In Scope:**
- Neues Enum `GateType` mit den 7 Gates: SPREAD, VOLUME, RSI, EMA_TREND, VWAP, ATR, ADX
- Neues Record `GateCascadeResult` mit Liste von Einzel-Gate-Resultaten und Gesamt-Pass/Fail
- Neues Record `GateEvaluation` (pro Gate: GateType, passed, actualValue, threshold, reason)
- Integration in `QuantScore` oder als Ersatz (QuantScore wird zum Wrapper ueber GateCascadeResult)

**Out of Scope:**
- Gate-Evaluierungs-Logik (Short-Circuit, Schwellenberechnung) -- kommt in ODIN-011 (brain)
- Aenderung an `QuantScore`-Berechnung selbst -- kann parallel existieren und spaeter migriert werden
- Persistierung von Gate-Ergebnissen

## Acceptance Criteria

- [ ] Enum `GateType` mit 7 Werten und definierten Reihenfolge-Ordinal
- [ ] Record `GateEvaluation(GateType gate, boolean passed, double actualValue, double threshold, String reason)`
- [ ] Record `GateCascadeResult(List<GateEvaluation> evaluations, boolean allPassed, GateType firstFailedGate)`
- [ ] `firstFailedGate` ist null wenn alle Gates bestanden
- [ ] Evaluations-Liste hat feste Reihenfolge (Gate-Ordinal)

## Technical Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/model/GateType.java`
- `odin-api/src/main/java/de/its/odin/api/dto/GateEvaluation.java`
- `odin-api/src/main/java/de/its/odin/api/dto/GateCascadeResult.java`

## Concept References

- `docs/concept/03-strategy-logic.md` -- Abschnitt 4.2 "Gate-Kaskade (7 Hard Gates)", Tabelle "Gates mit Schwellen"
- `docs/concept/09-backtesting-evaluation.md` -- Abschnitt 11 "Gate-Kaskade fuer Decision Logic"

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
- [ ] Records fuer DTOs (`GateEvaluation`, `GateCascadeResult` als Records)
- [ ] ENUM statt String fuer endliche Mengen (`GateType` als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Keine Abhaengigkeit ausser JDK + Validation-API (odin-api Regel)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: `GateCascadeResult` mit allen Gates bestanden -- `allPassed=true`, `firstFailedGate=null`
- [ ] Unit-Test: `GateCascadeResult` mit erstem Gate gefailed -- `allPassed=false`, `firstFailedGate=SPREAD`
- [ ] Unit-Test: Evaluations-Liste hat korrekte Reihenfolge (Gate-Ordinal)
- [ ] Unit-Test: `GateType`-Ordinals entsprechen der normativen Gate-Reihenfolge
- [ ] Testklassen-Namenskonvention: `GateCascadeResultTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendiger `GateCascadeResult` mit 7 `GateEvaluation`-Eintraegen korrekt aufgebaut
- [ ] Integrationstest: Mixed-Szenario -- einige Gates bestanden, eines gefailed, `firstFailedGate` korrekt gesetzt
- [ ] Testklassen-Namenskonvention: `GateCascadeIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- odin-api enthaelt keine Datenbankzugriffe.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `GateType.java`, `GateEvaluation.java`, `GateCascadeResult.java`, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. leere Evaluations-Liste, doppelter Gate-Typ, `actualValue` = `threshold` exakt, alle Gates bestanden aber `allPassed=false`)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety bei `firstFailedGate`, Konsistenz zwischen `allPassed` und `evaluations`"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/03-strategy-logic.md` Abschnitt 4.2 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 7 Gates korrekt im Enum abgebildet sind und die Reihenfolge dem Konzept entspricht"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. was passiert wenn ein Gate-Typ in der Liste fehlt, Unveraenderlichkeit der Evaluations-Liste, Gleichheitssemantik bei `GateCascadeResult`?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. ob `GateCascadeResult` Factory-Methoden braucht)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Die Gate-Reihenfolge ist normativ: SPREAD -> VOLUME -> RSI -> EMA_TREND -> VWAP -> ATR -> ADX
- Short-Circuit-Logik (fruehes Abbrechen bei erstem Fail) wird in ODIN-011 (brain) implementiert, hier nur das Modell
- Bestehender `QuantScore` muss nicht sofort geaendert werden -- kann parallel existieren und spaeter migriert werden
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
