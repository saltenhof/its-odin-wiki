# ODIN-034: Challenger Suite (20 Scenarios)

**Modul:** odin-backtest
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-035 (Governance Variants)
**Geschaetzter Umfang:** L

---

## Kontext

Das Konzept definiert 20 Challenger-Szenarien (S01-S20) die systematisch verschiedene Marktbedingungen testen. Der aktuelle Backtest-Runner fuehrt nur einfache Backtests aus. Die Challenger Suite ist das Regressions-Framework das sicherstellt, dass Strategie-Aenderungen keine unbeabsichtigten Verschlechterungen verursachen. Sie fungiert als Release-Gate.

## Scope

**In Scope:**
- Szenario-Katalog mit 20 Szenarien (definiert in Konzept 09, Abschnitt 14) als Enum oder Konfiguration
- `ChallengerSuiteRunner`: Fuehrt alle 20 Szenarien sequenziell oder parallel aus
- Ergebnisvergleich: Aktuelle Strategie vs. Baseline
- Pass/Fail-Kriterien pro Szenario
- Gesamtergebnis: `ChallengerSuiteReport` mit pro-Szenario-Ergebnis und Gesamt-Verdict

**Out of Scope:**
- Automatischer CI/CD-Trigger (manueller Aufruf in V1)
- Visuelle Darstellung des Reports im Frontend
- Mehr als 20 Szenarien (Erweiterbarkeit muss aber moeglich sein)

## Akzeptanzkriterien

- [ ] 20 Szenarien definiert als Enum oder Konfiguration mit Beschreibung und erwartetes Verhalten
- [ ] Szenarien decken ab: Trend Days (S01-S04), Range Days (S05-S08), Volatile Days (S09-S12), Edge Cases (S13-S16), Stress (S17-S20)
- [ ] Jedes Szenario hat: Beschreibung, erwartetes Verhalten, Pass/Fail-Kriterium
- [ ] `ChallengerSuiteRunner` fuehrt alle 20 Szenarien sequenziell oder parallel aus
- [ ] `ChallengerSuiteReport` Record mit: pro-Szenario-Ergebnis (PASS/FAIL + Metriken) und Gesamt-Verdict
- [ ] Release-Gate: Alle 20 muessen bestanden sein (oder konfigurierbare Mindest-Pass-Rate)
- [ ] Suite-Runner-Ergebnis ist deterministisch (gleiche Eingabe = gleiche Ausgabe)

## Technische Details

**Dateien:**
- `odin-backtest/src/main/java/de/its/odin/backtest/challenger/ChallengerScenario.java` (Enum oder Record)
- `odin-backtest/src/main/java/de/its/odin/backtest/challenger/ChallengerSuiteRunner.java` (neue Klasse)
- `odin-backtest/src/main/java/de/its/odin/backtest/challenger/ChallengerSuiteReport.java` (neues Record)

## Konzept-Referenzen

- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 14 "20 Challenger-Szenarien (S01-S20)"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 14, Tabelle "20 Szenarien mit Beschreibung"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 12 "Release-Gate Pipeline"
- `docs/concept/11-edge-cases.md` — Abschnitt 7 "Challenger Suite Coverage Mapping"

## Guardrail-Referenzen

- `docs/backend/guardrails/development-process.md` — "Build Gates: Challenger Suite"
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Namensregeln
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln, ENUM statt String fuer endliche Mengen

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-backtest`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `DEFAULT_MIN_PASS_RATE`, `SCENARIO_COUNT`)
- [ ] Records fuer DTOs (`ChallengerSuiteReport`)
- [ ] ENUM fuer `ChallengerScenario` (endliche Menge von 20 Szenarien)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.backtest.challenger.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `ChallengerSuiteRunner`: Suite-Runner mit Mock-Backtests (alle PASS, Mix PASS/FAIL, alle FAIL)
- [ ] Unit-Tests fuer Pass/Fail-Logik: Release-Gate greift korrekt bei konfigurierter Mindest-Pass-Rate
- [ ] Unit-Tests: Alle 20 Szenarien korrekt definiert (Vollstaendigkeitspruefung)
- [ ] Unit-Tests: `ChallengerSuiteReport` enthaelt korrekte aggregierte Daten
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer BacktestRunner
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `ChallengerSuiteRunner` mit echtem (stub-artigen) `BacktestRunner` fuer mindestens 3 Szenarien End-to-End
- [ ] Integrationstest: Gesamt-Verdict korrekt aus Einzel-Ergebnissen abgeleitet
- [ ] Integrationstest: Konfigurierbare Mindest-Pass-Rate wird korrekt angewendet
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass Suite-Runner, Szenario-Definitionen und Report-Generierung korrekt zusammenarbeiten.

### 2.4 Tests — Datenbank

Nicht zutreffend. `ChallengerSuiteRunner` ist eine reine Orchestrierungsklasse ohne direkten Datenbankzugriff. Persistenz liegt beim delegierten `BacktestRunner`.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `ChallengerSuiteRunner.java`, `ChallengerScenario.java` (alle 20 Szenarien), `ChallengerSuiteReport.java`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn ein Szenario keine Testdaten hat? Wie mit NaN-Metriken umgehen? Wie sicherstellen dass Szenarien repraesentativ sind?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, korrekte Pass/Fail-Aggregation, Null-Safety bei fehlenden Szenario-Ergebnissen, Determinismus-Garant"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/09-backtesting-evaluation.md` Abschnitte 12 und 14 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 20 Szenarien korrekt aus dem Konzept abgeleitet sind. Stimmen Szenario-Kategorien, Pass/Fail-Kriterien und Release-Gate-Logik mit Konzept ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Welche praktischen Probleme entstehen bei der Pflege einer Challenger Suite ueber Zeit? Was passiert wenn Marktregimes sich strukturell aendern und historische Szenarien nicht mehr repraesentativ sind?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Enum vs. Konfiguration fuer Szenarien, Sequenziell vs. Parallel)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- Die 20 Szenarien stehen detailliert in Konzept 09, Abschnitt 14 — dort die genauen Definitionen entnehmen
- Jedes Szenario braucht spezifische Testdaten (bestimmte Tage mit bekanntem Marktverhalten)
- Szenarien koennen anfangs mit synthetischen Daten betrieben werden, spaeter mit realen historischen Daten
- ODIN-035 (Governance Variants) muss vor dieser Story abgeschlossen sein — jedes Szenario sollte optional mit einer GovernanceVariant parametrisiert werden koennen
