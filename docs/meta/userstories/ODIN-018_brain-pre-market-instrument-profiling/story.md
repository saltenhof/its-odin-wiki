# ODIN-018: Pre-Market Instrument Profiling

**Modul:** odin-brain
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-017 (LLM Tactical Parameter Controller)

---

## Kontext

Das Konzept definiert eine Pre-Market-LLM-Analyse waehrend der WARMUP-Phase. Das LLM analysiert historische Muster des Instruments und liefert ein "Instrument Profile" mit 5 Dimensionen. Dieses Profil wird als Kontext fuer alle weiteren LLM-Calls des Tages verwendet.

## Scope

**In Scope:**
- Neuer LLM-Call-Typ: Instrument Profiling (waehrend WARMUP)
- 5 Profil-Dimensionen: Volatility-Regime, Typical-Range, Trend-Persistence, Mean-Reversion-Tendency, News-Sensitivity
- Ergebnis als `InstrumentProfile` Record im API-Modul
- Integration in PromptBuilder als Basis-Kontext fuer alle Intraday-Calls
- Fallback bei LLM-Fehler: Default-Profil (kein Blockieren des Tagesstarts)

**Out of Scope:**
- Intraday-Profil-Updates (Profil wird einmalig waehrend WARMUP erstellt, kein Re-Call)
- Speicherung des Profils in der Datenbank ueber den Tag hinaus
- Profil-Vergleich ueber mehrere Handelstage (historische Analyse)

## Akzeptanzkriterien

- [ ] Neues Record `InstrumentProfile` mit 5 Dimensionen (jeweils als Score 0.0-1.0 oder Enum)
- [ ] LLM-Call wird einmalig in der WARMUP-Phase ausgeloest
- [ ] Prompt enthaelt: Letzte 20 Daily Bars, Symbol-Name, Exchange
- [ ] Profil wird gecacht fuer den gesamten Tag (kein Re-Call)
- [ ] Bei LLM-Fehler: Default-Profil mit mittleren Werten (kein Absturz, kein Tag-Blockieren)
- [ ] Profil fliesst in den Context-Prompt aller weiteren LLM-Calls ein (via PromptBuilder)
- [ ] EventLog-Eintrag: INSTRUMENT_PROFILE mit den 5 Dimensionen

## Technische Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/dto/InstrumentProfile.java` (neues Record)
- `odin-brain/src/main/java/de/its/odin/brain/llm/InstrumentProfiler.java` (neue Klasse, fuehrt den LLM-Call durch)
- `odin-brain/src/main/java/de/its/odin/brain/llm/PromptBuilder.java` (Erweiterung: Profil-Kontext einfuegen)

**Integrationspunkt:** `InstrumentProfiler` wird vom Pipeline-Lifecycle waehrend WARMUP aufgerufen. Das Ergebnis wird in der Pipeline-Session gecacht.

## Konzept-Referenzen

- `docs/concept/04-llm-integration.md` -- Abschnitt 5 "Pre-Market Instrument Profiling"
- `docs/concept/04-llm-integration.md` -- Abschnitt 5, Tabelle "5 Profil-Dimensionen"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 1 "Pre-Market Phase"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "LLM Integration: Special Scenario"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api,odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten (DEFAULT_SCORE = 0.5, DAILY_BAR_COUNT = 20)
- [ ] Records fuer DTOs (`InstrumentProfile` als Java Record)
- [ ] ENUM statt String fuer endliche Mengen (falls Dimensionen als Enum statt Score)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.llm.profiling.*`
- [ ] Port-Abstraktion: Gegen `LlmAnalyst`-Interface aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Profil-Parsing -- valide LLM-Response -> InstrumentProfile korrekt befuellt
- [ ] Unit-Test: Default-Fallback -- LLM-Fehler -> Default-Profil mit Score 0.5 fuer alle Dimensionen
- [ ] Unit-Test: Caching -- zweiter Aufruf von InstrumentProfiler gibt gecachtes Ergebnis zurueck (kein zweiter LLM-Call)
- [ ] Unit-Test: Prompt-Inhalt -- Daily-Bars-Daten, Symbol-Name und Exchange korrekt im Prompt enthalten
- [ ] Unit-Test: EventLog-Eintrag -- INSTRUMENT_PROFILE wird geloggt mit allen 5 Dimensionen
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer LLM-Client und EventLog (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `InstrumentProfiler` zusammen mit `PromptBuilder` (reale Klassen)
- [ ] Integrationstest: Profil-Inhalt fliesst korrekt in Context-Prompt ein (PromptBuilder gibt Profil-Daten aus)
- [ ] Mindestens 1 Integrationstest der den Pfad WARMUP-Phase -> InstrumentProfiler -> gecachtes Profil -> PromptBuilder-Kontext durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- das Profil wird nur in-memory gecacht, keine DB-Persistenz in dieser Story.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: InstrumentProfiler-Code, InstrumentProfile-Record, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei teilweiser LLM-Response (nur 3 von 5 Dimensionen)? Was wenn die letzten 20 Daily Bars nicht verfuegbar sind (neues Symbol)? Thread-Safety des Caches?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Null-Safety bei LLM-Response-Feldern, Cache-Thread-Safety, Fallback-Logik bei partieller Response, Ressource-Leaks"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/04-llm-integration.md` Abschnitt 5 und `docs/concept/03-strategy-logic.md` Abschnitt 1 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche 5 Profil-Dimensionen, Caching-Semantik, Fallback-Verhalten, WARMUP-Phase-Integration"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Was passiert wenn das Pre-Market-Profiling sehr lange dauert (LLM-Timeout)? Sollte das Profil bei extremen Intraday-Ereignissen (Circuit Breaker, Halted Trading) neu erstellt werden?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Cache-Implementierung, Fallback-Score-Werte, Prompt-Format fuer Daily-Bars)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Das Profiling geschieht VOR dem ersten Trading-LLM-Call und ist kein Blocking-Call fuer den Tagesstart
- Wenn das LLM-Profiling fehlschlaegt, startet der Tag trotzdem (Default-Profil)
- Die 5 Dimensionen sind im Konzept 04, Abschnitt 5 detailliert beschrieben
- Cache-Thread-Safety: Das gecachte InstrumentProfile wird nur einmal geschrieben und dann nur gelesen -- AtomicReference reicht aus
- Abhaengigkeit zu ODIN-017: PromptBuilder muss das Profil als Kontext einbinden -- das setzt ODIN-017 (neuer PromptBuilder) voraus
