# ODIN-013: Setup D (Re-Entry after Correction) Pattern FSM

**Modul:** odin-brain
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-009 (PatternFeatures)

---

## Kontext

Setup D (Re-Entry after Correction) fehlt in der Codebase. Dieses Pattern erkennt einen intakten Trend, der kurzzeitig korrigiert (Pullback zu Support-Level wie EMA, VWAP, oder Fibonacci-Retracement), und signalisiert Re-Entry bei Wiederaufnahme der Trendrichtung.

## Scope

**In Scope:**
- Neue Klasse `ReEntryCorrectionFsm` implementiert `PatternStateMachine`
- 4 FSM-States: IDLE, TREND_ESTABLISHED, PULLBACK_ACTIVE, SIGNAL_ACTIVE
- Erkennung: Trend vorhanden (ADX > 25), Pullback zu Support (EMA50, VWAP, 50% Retracement), Wiederaufnahme (Close > Pullback-High)
- Confidence-Berechnung basierend auf Pullback-Tiefe

**Out of Scope:**
- Multi-Cycle Re-Entry (ODIN-025) -- Setup D ist INNERHALB eines Cycles
- Aenderungen an bestehenden FSMs
- Aenderungen am PatternStateMachine-Interface

## Akzeptanzkriterien

- [ ] `ReEntryCorrectionFsm` implementiert `PatternStateMachine` Interface
- [ ] IDLE -> TREND_ESTABLISHED: ADX > 25 + EMA50 steigend + mindestens 3 Higher-Lows
- [ ] TREND_ESTABLISHED -> PULLBACK_ACTIVE: Preis faellt zu Support-Level (EMA50 oder VWAP oder 38.2%-61.8% Retracement)
- [ ] PULLBACK_ACTIVE -> SIGNAL_ACTIVE: Close > letztes Swing-High VOR dem Pullback oder Close > EMA9 (kurzfristig)
- [ ] Timeout: Wenn Pullback > 3 ATR tief wird -> Trend gebrochen -> IDLE
- [ ] PatternSignal: entryPrice (aktueller Preis), stopLevel (Pullback-Low), confidence (basierend auf Pullback-Tiefe)
- [ ] Confidence hoeher bei 50% Retracement als bei 61.8% (flacherer Pullback = staerkerer Trend)
- [ ] Konfigurierbare Parameter in `BrainProperties.PatternProperties`
- [ ] Registration in `RulesEngine.initializePatternFsms()`

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/rules/pattern/ReEntryCorrectionFsm.java` (neue Klasse)
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` (Erweiterung PatternProperties)

**Muster:** Vergleiche existierende FSMs (FlushReclaimRunFsm, CoilBreakoutFsm) fuer das strukturelle Pattern.

## Konzept-Referenzen

- `docs/concept/03-strategy-logic.md` -- Abschnitt 6.4 "Setup D: Re-Entry nach Korrektur"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 6.4, FSM-Diagramm
- `docs/concept/03-strategy-logic.md` -- Abschnitt 7.4 "Re-Entry vs. Add"

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
- [ ] Keine Magic Numbers -- `private static final` Konstanten (Fibonacci-Levels als Konstanten)
- [ ] Records fuer DTOs (PatternSignal)
- [ ] ENUM statt String fuer endliche Mengen (FSM-States als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.rules.pattern.*` fuer Pattern-Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Vollstaendiger FSM-Durchlauf IDLE -> TREND_ESTABLISHED -> PULLBACK_ACTIVE -> SIGNAL_ACTIVE
- [ ] Unit-Test: Zu tiefer Pullback (> 3 ATR) -> kein Signal, Rueckkehr zu IDLE (Trend gebrochen)
- [ ] Unit-Test: Kein Trend (ADX < 25) -> kein Uebergang zu TREND_ESTABLISHED
- [ ] Unit-Test: Confidence-Berechnung -- 50% Retracement ergibt hoehere Confidence als 61.8%
- [ ] Unit-Test: Nicht genug Higher-Lows (< 3) -> kein Uebergang zu TREND_ESTABLISHED
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer PatternFeatures (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `ReEntryCorrectionFsm` eingebettet in `RulesEngine` (reale Klassen zusammengeschaltet)
- [ ] Integrationstest: Vollstaendiger Markt-Tick-Strom mit Trend-Pullback-Wiederaufnahme -> FSM-Signalausgabe
- [ ] Mindestens 1 Integrationstest der den Pfad Tick-Daten -> FSM -> PatternSignal durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: FSM-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei exakt 61.8% Retracement? Was bei mehreren aufeinanderfolgenden Pullbacks? Wie wird Swing-High bei volatilen Maerkten bestimmt?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, fehlerhafte State-Transitionen, Fibonacci-Berechnung, Null-Safety, Higher-Low-Erkennungslogik"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/03-strategy-logic.md` Abschnitt 6.4 und 7.4 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die FSM-Implementierung dem Konzept entspricht. Vergleiche State-Transitionen, Fibonacci-Levels, Timeout-Bedingung, Unterschied zu Multi-Cycle Re-Entry"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Abgrenzung zu Setup B bei V-Recovery-Mustern? Was passiert wenn VWAP und EMA50 sehr nah beieinander sind (welcher Support gewinnt)?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Fibonacci-Implementierung, Swing-High-Berechnung, Confidence-Formel)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Fibonacci-Retracement: 38.2%, 50%, 61.8% vom letzten Swing-Low zum Swing-High
- Pullback-Tiefe muss begrenzt sein: > 61.8% Retracement bedeutet wahrscheinlich Trend-Bruch
- Dieses Setup ist auch in Multi-Cycle relevant (ODIN-025), aber die FSM selbst kennt keine Cycles
- Fibonacci-Levels als `private static final` Konstanten (nicht Magic Numbers): 0.382, 0.500, 0.618
- Abgrenzung zu Multi-Cycle (ODIN-025): Setup D = Add-Position innerhalb laufendem Cycle. Multi-Cycle = Neueinstieg nach Flat
