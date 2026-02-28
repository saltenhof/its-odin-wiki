# ODIN-002: LLM Tactical Parameter Schema im API-Modul

**Modul:** odin-api
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-001 (Subregime-Enum)

---

## Context

Das aktuelle `LlmAnalysis`-Record ist ein "Analyst"-Schema (Regime, Confidence, Target, Duration). Das Konzept definiert das LLM als "Tactical Parameter Controller" mit einem voellig neuen Feld-Schema: Decision-Felder, Context-Felder, Tactical P0-P3, und Logging-only Felder. Dieses neue Schema muss im API-Modul als Record(s) definiert werden.

## Scope

**In Scope:**
- Neues Record `LlmTacticalOutput` in `de.its.odin.api.event` mit allen Feldern gemaess Konzept 04, Abschnitt 2
- Neue Enums fuer bounded LLM-Felder: `ExitBias`, `TrailMode`, `ProfitProtectionProfile`, `TargetPolicy`, `EntryTiming`, `SizeModifier`
- `LlmAnalysis` bleibt bestehen fuer Backward-Compatibility (Backtest-Modus), aber wird als deprecated markiert
- Validierungs-Annotationen auf den Feldern (Ranges, Not-Null)

**Out of Scope:**
- LLM-Client-Implementierung (ClaudeAnalystClient, OpenAiAnalystClient) -- kommt in anderen Stories
- JSON-Deserialisierung / Prompt-Engineering -- kommt in anderen Stories
- Migration von `LlmAnalysis` zu `LlmTacticalOutput` im laufenden System

## Acceptance Criteria

- [ ] Record `LlmTacticalOutput` mit Decision-Feldern: `action` (ENTER/HOLD/EXIT/NO_TRADE), `regimeAssessment` (Regime), `subregimeAssessment` (Subregime), `confidenceScore` (0.0-1.0)
- [ ] Context-Felder: `marketNarrative` (String, max 500 chars), `keyLevels` (List<Double>), `catalysts` (List<String>)
- [ ] Tactical P0: `exitBias` (TRAIL_TIGHT/TRAIL_NORMAL/TRAIL_WIDE/HOLD_RUNNERS), `trailMode` (EMA_HL/ATR/STRUCTURE)
- [ ] Tactical P1: `profitProtectionProfile` (AGGRESSIVE/MODERATE/RELAXED), `targetPolicy` (FIXED_TARGETS/TRAIL_ONLY/HYBRID)
- [ ] Tactical P2: `entryTiming` (IMMEDIATE/WAIT_PULLBACK/WAIT_CONFIRMATION), `sizeModifier` (FULL/REDUCED/MINIMAL)
- [ ] Tactical P3: `holdDurationBars` (Integer), `reentryAllowed` (boolean)
- [ ] Logging-only: `reasoning` (String), `alternativeScenario` (String)
- [ ] Alle Enums als separate Dateien in `de.its.odin.api.model`
- [ ] `LlmAnalysis` mit `@Deprecated` annotiert und JavaDoc-Hinweis auf `LlmTacticalOutput`

## Technical Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/event/LlmTacticalOutput.java`
- `odin-api/src/main/java/de/its/odin/api/model/ExitBias.java`
- `odin-api/src/main/java/de/its/odin/api/model/TrailMode.java`
- `odin-api/src/main/java/de/its/odin/api/model/ProfitProtectionProfile.java`
- `odin-api/src/main/java/de/its/odin/api/model/TargetPolicy.java`
- `odin-api/src/main/java/de/its/odin/api/model/EntryTiming.java`
- `odin-api/src/main/java/de/its/odin/api/model/SizeModifier.java`
- `odin-api/src/main/java/de/its/odin/api/model/LlmAction.java`

Felder die NICHT im LLM-Output sein duerfen (Konzept 04, Abschnitt 2.2): Preise, Stueckzahlen, Order-Typen. Das LLM liefert nur bounded Enums und Scores.

## Concept References

- `docs/concept/04-llm-integration.md` -- Abschnitt 2 "Vollstaendige Feld-Schema"
- `docs/concept/04-llm-integration.md` -- Abschnitt 2.2 "Was das LLM NICHT liefert"
- `docs/concept/04-llm-integration.md` -- Abschnitt 3 "Tactical Decision Features"
- `docs/concept/08-symmetric-hybrid-protocol.md` -- Abschnitt 5 "JSON-Schema-Beispiele: LlmOutput"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "odin-api: Records fuer DTOs"
- `docs/backend/guardrails/module-structure.md` -- "ENUM statt String fuer endliche Mengen"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- "Events sind immutable"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (`LlmTacticalOutput` als Record)
- [ ] ENUM statt String fuer endliche Mengen (alle 6 Taktik-Enums als separate Enum-Dateien)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Keine Abhaengigkeit ausser JDK + Validation-API (odin-api Regel)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer Record-Validierung: ungueltige `confidenceScore`-Ranges (< 0.0, > 1.0)
- [ ] Unit-Tests: `LlmTacticalOutput` mit allen Pflichtfeldern korrekt konstruiert
- [ ] Unit-Tests: Enum-Vollstaendigkeit fuer alle 6 Taktik-Enums
- [ ] Testklassen-Namenskonvention: `LlmTacticalOutputTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `LlmTacticalOutput` vollstaendig befuellt, alle Felder korrekt gesetzt und abrufbar
- [ ] Integrationstest: Null-Safety -- nullable Felder (Logging-only) koennen null sein ohne Fehler
- [ ] Integrationstest: `subregimeAssessment.parentRegime()` stimmt mit `regimeAssessment` ueberein
- [ ] Testklassen-Namenskonvention: `LlmTacticalOutputIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- odin-api enthaelt keine Datenbankzugriffe.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `LlmTacticalOutput.java`, alle Enum-Dateien, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. inkonsistente regimeAssessment/subregimeAssessment, confidenceScore=0.0, leere keyLevels-Liste, sehr langer marketNarrative-String)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, Validierungsluecken, fehlende Enum-Abdeckung"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/04-llm-integration.md` Abschnitte 2, 2.2, 3 + `docs/concept/08-symmetric-hybrid-protocol.md` Abschnitt 5 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob das Schema vollstaendig dem Konzept entspricht. Vergleiche alle Felder, Enum-Werte, und was das LLM NICHT liefern darf"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Versionierung des Schemas, JSON-Deserialisierungs-Fallstricke, Verhalten bei partiell befuelltem LLM-Output?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. warum `LlmAction` als separates Enum statt direkt im Record)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Konzept 04 Abschnitt 2 ist die primaere Quelle fuer das Schema. Abschnitt 5 in 08 hat JSON-Beispiele
- Die Enums muessen bounded sein -- kein freier String. Das ist die zentrale Anti-Halluzination-Massnahme
- `LlmAnalysis` NICHT loeschen -- wird fuer Backtest-Modus (CachedAnalyst) noch gebraucht. Langfristig Migration
- Beachte: `action` im LLM-Output hat andere Semantik als `IntentType` im TradeIntent. Das LLM "empfiehlt", der Arbiter "entscheidet"
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
