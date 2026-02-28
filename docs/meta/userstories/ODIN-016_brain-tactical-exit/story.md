# ODIN-016: TACTICAL_EXIT Mechanism

**Modul:** odin-brain
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** ODIN-015 (Exhaustion Detection), ODIN-002 (LLM Tactical Schema)

---

## Kontext

Das Konzept definiert TACTICAL_EXIT als LLM-getriggerten Exit der nur unter bestimmten Bedingungen erlaubt ist (Mindestgewinn, Exhaustion-Bestaetigung). Es ist ein "kontrollierter fruehzeitiger Ausstieg" wenn das LLM Schwaeche erkennt, bevor der mechanische Trail greift.

## Scope

**In Scope:**
- TACTICAL_EXIT als neuer Exit-Pfad in `ExitRules`
- Bedingungen: LLM `exitBias` == HOLD_RUNNERS ist NICHT gesetzt + LLM signalisiert EXIT + aktuelle Position im Gewinn (> 0.5R)
- Bestaetigung durch Exhaustion Detection (mindestens 1 Pillar) oder Regime-Confidence < Schwelle
- Anti-LLM-Drift: TACTICAL_EXIT maximal 1x pro Position, Mindest-Haltedauer 3 Bars

**Out of Scope:**
- Aenderungen am ExhaustionDetector selbst (ODIN-015)
- Neue Exit-Reasons ausserhalb von TACTICAL_EXIT
- Order-Ausfuehrungslogik (TACTICAL_EXIT ist Exit-Intent an Arbiter, nicht direkte Order)

## Akzeptanzkriterien

- [ ] TACTICAL_EXIT wird nur geprueft wenn LLM-Analysis vorhanden und `action == EXIT`
- [ ] Mindestgewinn: Position muss > 0.5R im Plus sein
- [ ] Bestaetigung: Mindestens 1 Exhaustion-Pillar ODER regime_confidence < 0.3
- [ ] Anti-Drift: Maximal 1 TACTICAL_EXIT pro Position (weiterer Versuch wird blockiert)
- [ ] Mindest-Haltedauer: Position muss mindestens 3 Decision-Bars gehalten worden sein
- [ ] ExitReason ist `TACTICAL_EXIT`
- [ ] Log-Eintrag mit LLM-Reasoning und Bestaetigung (welcher Pillar aktiv, oder Confidence-Wert)

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/rules/ExitRules.java` (Erweiterung: TACTICAL_EXIT-Pfad)
- `odin-api/src/main/java/de/its/odin/api/model/ExitReason.java` (falls TACTICAL_EXIT noch nicht von ODIN-015 hinzugefuegt)

**Prioritaet in ExitRules:** nach TRAILING_STOP, vor REGIME_CHANGE.

## Konzept-Referenzen

- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 7 "TACTICAL_EXIT-Mechanismus"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 7, Bedingungstabelle
- `docs/concept/04-llm-integration.md` -- Abschnitt 8 "Anti-LLM-Drift Guardrails fuer Stop/Trail"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-api`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten (MIN_GAIN_R = 0.5, MIN_HOLD_BARS = 3, MAX_CONFIDENCE_THRESHOLD = 0.3)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention eingehalten
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: TACTICAL_EXIT tritt ein wenn alle Bedingungen erfuellt (LLM EXIT + > 0.5R + Pillar aktiv + >= 3 Bars)
- [ ] Unit-Test: Kein TACTICAL_EXIT wenn Position < 0.5R (Mindestgewinn nicht erreicht)
- [ ] Unit-Test: Kein TACTICAL_EXIT wenn keine Exhaustion-Bestaetigung (0 Pillars + Confidence >= 0.3)
- [ ] Unit-Test: Anti-Drift -- 2. TACTICAL_EXIT-Versuch in derselben Position wird blockiert
- [ ] Unit-Test: Mindest-Haltedauer -- Position mit nur 2 Decision-Bars -> kein TACTICAL_EXIT
- [ ] Unit-Test: HOLD_RUNNERS gesetzt -> TACTICAL_EXIT wird nicht ausgeloest
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer ExhaustionDetector, LlmAnalysis, Position-State (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `ExitRules` mit realem `ExhaustionDetector` und simulierter LLM-Analyse (reale Klassen)
- [ ] Integrationstest: Vollstaendiger Exit-Entscheidungspfad -- LLM-EXIT-Signal + 2 aktive Pillars -> TACTICAL_EXIT produziert
- [ ] Mindestens 1 Integrationstest der die korrekte Prioritaet (nach TRAILING_STOP, vor REGIME_CHANGE) in ExitRules bestaetigt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: ExitRules-Erweiterung (TACTICAL_EXIT-Pfad), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn LLM-Analysis ablaeuft (Freshness-Gate) waehrend TACTICAL_EXIT geprueft wird? Was wenn die Position genau bei 0.5R ist (Grenzwert)? Anti-Drift-Counter bei Position-Reset?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Anti-Drift-Counter-Reset-Logik, Null-Safety bei LLM-Analysis-Feldern, Race Conditions bei Bar-Zaehlung"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/05-stops-profit-protection.md` Abschnitt 7 und `docs/concept/04-llm-integration.md` Abschnitt 8 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche alle Bedingungen, Anti-Drift-Logik, HOLD_RUNNERS-Interaktion, Prioritaet in ExitRules"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Was passiert wenn TACTICAL_EXIT ausgeloest wird aber der Broker die Order ablehnt -- wird Anti-Drift-Counter trotzdem gesetzt? Wie interagiert TACTICAL_EXIT mit Multi-Tranche-Positionen?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Anti-Drift-Counter-Verwaltung, HOLD_RUNNERS-Semantik)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- TACTICAL_EXIT ist KEIN Market-Order. Es wird als normaler Exit-Intent an den Arbiter weitergegeben
- Prioritaet in ExitRules: nach TRAILING_STOP, vor REGIME_CHANGE
- Das LLM kann TACTICAL_EXIT empfehlen, aber nur wenn die Bestaetigung vorhanden ist
- Anti-Drift-Counter muss beim Oeffnen einer neuen Position zurueckgesetzt werden
- Abhaengigkeit zu ODIN-015: ODIN-015 muss abgeschlossen sein (ExhaustionDetector vorhanden) bevor diese Story gestartet wird
