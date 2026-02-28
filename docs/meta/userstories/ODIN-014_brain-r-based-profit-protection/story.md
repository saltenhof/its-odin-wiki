# ODIN-014: R-basierte Profit Protection

**Modul:** odin-brain
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-002 (LLM Tactical Schema, insb. ProfitProtectionProfile)

---

## Kontext

Das Konzept definiert eine R-basierte (Risk-Multiple) Profit Protection mit einer gestuften Tabelle. Je hoeher der realisierte R-Wert, desto enger wird der minimale Stop. Die aktuelle ExitRules-Klasse hat nur einen einfachen ATR-basierten Trailing Stop. Die R-Tabelle fuegt eine "Floor"-Logik hinzu: Der hoechere Wert von Trail und R-Floor gewinnt.

## Scope

**In Scope:**
- Neue Klasse `ProfitProtectionEngine` in `de.its.odin.brain.rules`
- R-Stufen-Tabelle mit 6 Levels gemaess Konzept 05, Abschnitt 2
- MFE-Lock-Mechanismus (Maximum Favorable Excursion)
- Integration in `ExitRules`: R-Floor als Minimum-Stop, hoeherer Wert gewinnt gegen Trail
- 3 Profile (AGGRESSIVE, MODERATE, RELAXED) mit unterschiedlichen Tabellen-Parametern
- LLM kann Profil ueber `ProfitProtectionProfile` waehlen

**Out of Scope:**
- Aenderungen am Trailing-Stop-Mechanismus selbst (nur Floor-Logik)
- Neue R-Stufen ausserhalb der 6 definierten Levels
- Backtesting-Integration (folgt spaeter)

## Akzeptanzkriterien

- [ ] R-Berechnung: `currentR = (currentPrice - entryPrice) / (entryPrice - initialStop)`
- [ ] Stufe < 1R: Kein Profit-Protection-Floor (nur normaler Trail)
- [ ] Stufe 1.0-1.5R: Floor bei Break-Even (Entry-Preis)
- [ ] Stufe 1.5-2.0R: Floor bei Entry + 0.5R
- [ ] Stufe 2.0-3.0R: Floor bei Entry + 1.0R
- [ ] Stufe 3.0-4.0R: Floor bei Entry + 1.5R
- [ ] Stufe >= 4.0R: Floor bei Entry + 2.0R (aggressive Sicherung)
- [ ] MFE-Lock: Einmal erreichtes R-Level wird nie zurueckgesetzt (Highwater)
- [ ] Effektiver Stop = max(trailingStop, rFloor) -- hoeherer Wert gewinnt
- [ ] AGGRESSIVE Profil: Engere Stufen (z.B. Floor bei 1.5R schon bei 1.0R erreicht)
- [ ] RELAXED Profil: Weitere Stufen (z.B. Floor bei 1.5R erst bei 2.0R erreicht)
- [ ] MODERATE ist der Default
- [ ] Profil-Wechsel durch LLM ueber `ProfitProtectionProfile` moeglich

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/rules/ProfitProtectionEngine.java` (neue Klasse)
- `odin-brain/src/main/java/de/its/odin/brain/rules/ExitRules.java` (Integration: Floor-Logik)
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` (R-Tabelle, Profile)

**Schluessellogik:** `effectiveStop = Math.max(trailingStop, rFloor)`. Der R-Floor ist ein Highwater-Mark und darf nur steigen, nie fallen.

## Konzept-Referenzen

- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 2 "R-basierte Profit Protection"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 2, Tabelle "6 R-Stufen"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 3 "MFE-Lock-Mechanismus"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 4 "Regime-spezifische Trail-Breiten"
- `docs/concept/04-llm-integration.md` -- Abschnitt 3 "profit_protection_profile"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH fuer Trading-Logic"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- "Trailing Stop = Fallnetz, ATR Intraday einfrieren"

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten (alle R-Schwellen, z.B. `R_LEVEL_1 = 1.0`, `R_LEVEL_1_5 = 1.5`)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (ProfitProtectionProfile als Enum: AGGRESSIVE, MODERATE, RELAXED)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.rules.profit-protection.*`
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Jede R-Stufe einzeln (korrekte Floor-Berechnung bei < 1R, 1-1.5R, 1.5-2R, 2-3R, 3-4R, >= 4R)
- [ ] Unit-Test: MFE-Lock -- einmal 2R erreicht, Preis faellt zurueck auf 1.5R -> Floor bleibt bei 1.0R-Level (Highwater)
- [ ] Unit-Test: max(trail, rFloor) Logik -- Trail groesser als Floor: Trail gewinnt; Floor groesser: Floor gewinnt
- [ ] Unit-Test: Alle 3 Profile (AGGRESSIVE, MODERATE, RELAXED) -- unterschiedliche Stufen-Trigger
- [ ] Unit-Test: R-Floor darf NUR steigen, nie fallen (Highwater-Invariante)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer Position-State (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `ProfitProtectionEngine` zusammen mit `ExitRules` (reale Klassen)
- [ ] Integrationstest: Vollstaendiger Exit-Entscheidungspfad -- Position mit R-Gewinn -> korrekter effektiver Stop
- [ ] Mindestens 1 Integrationstest der den Pfad Tick-Daten -> ExitRules -> effektiver Stop durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff. Die R-Stufentabelle ist in-memory konfiguriert.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: ProfitProtectionEngine-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei exakt 1.0R (Grenzbedingung)? Was wenn initialStop == entryPrice (Division durch Null)? Profil-Wechsel WAEHREND laufender Position?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, moegliche Division-durch-Null, Highwater-Mark-Verletzungen, Floating-Point-Praezisionsprobleme bei R-Vergleichen"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/05-stops-profit-protection.md` Abschnitte 2-4 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche R-Stufen-Tabelle, MFE-Lock-Semantik, Profil-Unterschiede, max()-Logik"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Was passiert bei einem Profil-Wechsel durch das LLM waehrend laufender Position -- wird der Floor angepasst oder eingefroren? Was passiert bei Gaps (Preis springt ueber mehrere R-Stufen gleichzeitig)?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Umgang mit Division-durch-Null, Profil-Wechsel-Semantik)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- KRITISCH: Der R-Floor darf NUR steigen, nie fallen (Highwater-Prinzip analog zum Trailing Stop)
- Die R-Tabelle ist eine User-Stakeholder-Entscheidung und steht explizit in Konzept 05
- "Runner-Trail bleibt, Profit-Protection als Floor" (MEMORY.md) -- der bestehende EMA/HL-Trail fuer Runner bleibt, R-Floor ist der Mindest-Stop
- Die 3 Profile muessen konfigurierbar sein (BrainProperties) mit Default-Werten aus dem Konzept
- Division-durch-Null abfangen: initialStop darf nicht gleich entryPrice sein (wuerde R undefiniert machen)
- Floating-Point-Vergleiche bei R-Stufen: Epsilon-Toleranz erwagen oder BigDecimal fuer kritische Grenzen
