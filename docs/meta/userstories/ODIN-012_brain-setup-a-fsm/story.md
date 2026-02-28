# ODIN-012: Setup A (Opening Consolidation -> Trend) Pattern FSM

**Modul:** odin-brain
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-009 (PatternFeatures), ODIN-008 (SessionPhase)

---

## Kontext

Von den 4 definierten Setups (A-D) sind aktuell nur B (Flush-Reclaim-Run) und C (Coil-Breakout) als Pattern-FSMs implementiert. Setup A (Opening Consolidation -> Trend) fehlt. Dies ist ein haeufiges Intraday-Pattern in den ersten 30-60 Minuten nach RTH-Open.

## Scope

**In Scope:**
- Neue Klasse `OpeningConsolidationFsm` implementiert `PatternStateMachine`
- 4 FSM-States: IDLE, CONSOLIDATING, BREAKOUT_PENDING, SIGNAL_ACTIVE
- Erkennung: Preis konsolidiert in enger Range (< 1 ATR) nach Open, dann Breakout mit Volume
- Zeitfenster: Nur in den ersten 60 Minuten RTH (SessionPhase == RTH_OPENING)
- Entry-Signal bei Breakout ueber Consolidation-High mit erhoehtem Volume

**Out of Scope:**
- Aenderungen an bestehenden FSMs (FlushReclaimRunFsm, CoilBreakoutFsm)
- Aenderungen an PatternStateMachine-Interface
- Setup B, C oder D

## Akzeptanzkriterien

- [ ] `OpeningConsolidationFsm` implementiert `PatternStateMachine` Interface
- [ ] IDLE -> CONSOLIDATING: Range der letzten N Bars < consolidationRangeAtrFactor * ATR
- [ ] CONSOLIDATING -> BREAKOUT_PENDING: Preis bricht ueber Consolidation-High (+ Offset)
- [ ] BREAKOUT_PENDING -> SIGNAL_ACTIVE: Volume > volumeConfirmationMultiplier * avgVolume
- [ ] Timeout: Wenn nach 60 Minuten RTH keine Consolidation erkannt -> IDLE (kein Signal)
- [ ] PatternSignal enthaelt: entryPrice (Consolidation-High), stopLevel (Consolidation-Low), confidence
- [ ] Nur waehrend RTH_OPENING SessionPhase aktiv
- [ ] Konfigurierbare Parameter in `BrainProperties.PatternProperties`
- [ ] Registration in `RulesEngine.initializePatternFsms()`

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/rules/pattern/OpeningConsolidationFsm.java` (neue Klasse)
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` (Erweiterung PatternProperties)

**Muster:** Vergleiche existierende FSMs `FlushReclaimRunFsm` und `CoilBreakoutFsm` fuer das strukturelle Pattern.

## Konzept-Referenzen

- `docs/concept/03-strategy-logic.md` -- Abschnitt 6.1 "Setup A: Opening Consolidation -> Trend"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 6.1, FSM-Diagramm
- `docs/concept/03-strategy-logic.md` -- Abschnitt 6, Tabelle "4 Setups mit Eigenschaften"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH fuer Trading-Logic"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (PatternSignal)
- [ ] ENUM statt String fuer endliche Mengen (FSM-States als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.rules.pattern.*` fuer Pattern-Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Vollstaendiger FSM-Durchlauf IDLE -> CONSOLIDATING -> BREAKOUT_PENDING -> SIGNAL_ACTIVE
- [ ] Unit-Test: Timeout nach 60 Minuten RTH -> kein Signal, Rueckkehr zu IDLE
- [ ] Unit-Test: Breakout ohne Volume-Bestaetigung -> SIGNAL_ACTIVE wird NICHT erreicht (verbleibt in BREAKOUT_PENDING)
- [ ] Unit-Test: Aktivierung nur bei SessionPhase == RTH_OPENING (Pre-Market oder After-Hours -> kein Signal)
- [ ] Unit-Test: Consolidation-Range zu gross (> 1 ATR) -> kein Uebergang zu CONSOLIDATING
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer PatternFeatures und SessionPhase (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `OpeningConsolidationFsm` eingebettet in `RulesEngine` (reale Klassen zusammengeschaltet)
- [ ] Integrationstest: Vollstaendiger Markt-Tick-Strom -> FSM-Signalausgabe End-to-End
- [ ] Mindestens 1 Integrationstest der den Pfad Tick-Daten -> FSM -> PatternSignal durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: FSM-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn die Session waehrend CONSOLIDATING wechselt? Was wenn der Preis zurueck unter Consolidation-Low faellt nach BREAKOUT_PENDING? Concurrent-State-Aenderungen?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, fehlerhafte State-Transitionen, Null-Safety bei Snapshot-Feldern, off-by-one Fehler bei Bar-Zaehlung"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/03-strategy-logic.md` Abschnitt 6.1 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die FSM-Implementierung dem Konzept entspricht. Vergleiche State-Transitionen, Bedingungen, Timeout-Logik, PatternSignal-Felder"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Was passiert wenn mehrere Consolidation-Phasen hintereinander erkannt werden? Wie verhalt sich die FSM bei duennem Pre-Market-Volumen direkt nach Open?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Consolidation-Range-Berechnung, Volume-Baseline)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Vergleiche die existierenden FSMs (FlushReclaimRunFsm, CoilBreakoutFsm) fuer das strukturelle Pattern
- Consolidation-Range: High-Low der letzten 10-15 1m-Bars (oder 3-5 3m-Bars)
- Entry bei Breakout ueber Consolidation-High + kleiner Offset (z.B. 0.1 ATR)
- Stop unter Consolidation-Low
- Dieses Setup ist zeitlich begrenzt -- nur in der Eroeffnungsphase relevant
- FSM-State ist pro Pipeline-Instanz (nicht global/shared)
