# ODIN-017: LLM als Tactical Parameter Controller

**Modul:** odin-brain
**Phase:** 1
**Groesse:** XL
**Abhaengigkeiten:** ODIN-002 (LLM Tactical Schema), ODIN-001 (Subregime)

---

## Kontext

Die aktuelle LLM-Integration (ClaudeProviderClient, OpenAiProviderClient, PromptBuilder) implementiert das "Analyst"-Paradigma: Das LLM liefert Regime + Confidence + Target. Das Konzept definiert ein neues Paradigma: "Tactical Parameter Controller". Das LLM liefert bounded Enums und taktische Parameter, keine Preise oder Stueckzahlen. Die gesamte Prompt-Architektur, Response-Parsing, und Validierung muss umgebaut werden.

## Scope

**In Scope:**
- Neuer 3-Tier Prompt-Aufbau: System-Prompt, Context-Prompt, Current-Tick-Prompt
- Neuer Response-Parser fuer `LlmTacticalOutput` Schema
- 5-Layer Anti-Halluzination: Schema-Validierung, Plausibility-Check, Consistency-Check, Freshness-Gate, Fallback
- Neuer CircuitBreaker fuer taktische Calls (separate Konfiguration)
- TTL/Freshness-Regeln: Entry-Freshness vs. Exit-Freshness vs. Monitoring-Freshness
- LLM-Call-Kadenz: Volatility-abhaengig + Event-getriggert
- Erweiterung PromptBuilder fuer den neuen Kontext (Indikatoren, Pattern-State, Position-State)

**Out of Scope:**
- Pre-Market Instrument Profiling (folgt in ODIN-018)
- Dual-Key Arbiter Integration (folgt in ODIN-019)
- Aenderungen am Broker-Adapter oder OMS

## Akzeptanzkriterien

- [ ] System-Prompt enthaelt: Rolle, Bounded-Enum-Definitions, Verbotene Felder, Response-Schema
- [ ] Context-Prompt enthaelt: Aktuelle Indikatoren (letzte 5 Bars), Regime/Subregime, Pattern-State, Position-State
- [ ] Current-Tick-Prompt: Aktueller Snapshot (Preis, Volume, Spread, VWAP)
- [ ] Response-Parsing: Strukturiertes JSON -> LlmTacticalOutput
- [ ] Schema-Validierung: Alle Felder vorhanden, Enums gueltig, Ranges eingehalten
- [ ] Plausibility-Check: Logische Konsistenz (z.B. EXIT + HOLD_RUNNERS ist Widerspruch)
- [ ] Consistency-Check: Keine extremen Aenderungen gegenueber letztem Call (Flip-Flop-Detection)
- [ ] Freshness: Maximale Gueltigkeit konfigurierbar (Default 180s fuer Entry, 300s fuer Exit)
- [ ] Kadenz: Normal alle 3 Bars, in HIGH_VOLATILITY jeden Bar, bei MonitorEvent sofort
- [ ] Bei Validierungsfehler: Fallback auf letzte gueltige Analyse (nicht crash)
- [ ] Claude und OpenAI Provider beide unterstuetzt
- [ ] Prompt-Versionierung: promptVersion Feld im Output

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/llm/PromptBuilder.java` (fundamentaler Umbau)
- `odin-brain/src/main/java/de/its/odin/brain/llm/LlmResponseParser.java` (neues Schema)
- `odin-brain/src/main/java/de/its/odin/brain/llm/SchemaValidator.java` (Erweiterung)
- `odin-brain/src/main/java/de/its/odin/brain/llm/PlausibilityChecker.java` (Erweiterung)
- `odin-brain/src/main/java/de/its/odin/brain/llm/ConsistencyChecker.java` (Erweiterung)
- `odin-brain/src/main/java/de/its/odin/brain/llm/LlmAnalystOrchestrator.java` (Kadenz-Logik)
- `odin-brain/src/main/java/de/its/odin/brain/llm/ClaudeProviderClient.java` (neues Schema)
- `odin-brain/src/main/java/de/its/odin/brain/llm/OpenAiProviderClient.java` (neues Schema)

**Hinweis:** Diese Story ist XL -- sie kann bei Bedarf in Sub-Stories gesplittet werden (Prompt + Parser + Validation + Kadenz). Das ist aber keine Pflicht, sondern bei Bedarf.

## Konzept-Referenzen

- `docs/concept/04-llm-integration.md` -- Abschnitt 1 "LLM als Tactical Parameter Controller"
- `docs/concept/04-llm-integration.md` -- Abschnitt 2 "Vollstaendige Feld-Schema"
- `docs/concept/04-llm-integration.md` -- Abschnitt 4 "LLM-Call-Kadenz"
- `docs/concept/04-llm-integration.md` -- Abschnitt 6 "3-Tier Prompt-Architektur"
- `docs/concept/04-llm-integration.md` -- Abschnitt 7 "5-Layer Anti-Halluzination"
- `docs/concept/04-llm-integration.md` -- Abschnitt 8 "TTL/Freshness-Regeln"
- `docs/concept/04-llm-integration.md` -- Abschnitt 10 "Anti-LLM-Drift Guardrails"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "LLM Integration: Special Scenario"
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13), LLM-Stack (Claude SDK + OpenAI API)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten (Freshness-Defaults, Kadenz-Intervalle, Consistency-Schwellen)
- [ ] Records fuer DTOs (LlmTacticalOutput als Record)
- [ ] ENUM statt String fuer endliche Mengen (alle bounded Enums aus dem Schema)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.llm.*` fuer LLM-Konfiguration
- [ ] Port-Abstraktion: Gegen `LlmAnalyst`-Interface aus `de.its.odin.api.port` programmieren
- [ ] LLM darf KEINE Preise, Stueckzahlen oder Order-Typen liefern -- nur bounded Enums

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Prompt-Aufbau -- System-Prompt enthaelt Bounded-Enum-Definitions und Response-Schema
- [ ] Unit-Test: Context-Prompt enthaelt Indikatoren, Regime/Subregime, Pattern-State, Position-State
- [ ] Unit-Test: Response-Parsing -- valides JSON -> LlmTacticalOutput korrekt geparst
- [ ] Unit-Test: Response-Parsing -- invalides JSON -> Fallback auf letzte gueltige Analyse (kein Crash)
- [ ] Unit-Test: Response-Parsing -- fehlende Pflichtfelder -> Validierungsfehler, kein Crash
- [ ] Unit-Test: Schema-Validierung -- ungueltige Enum-Werte -> Fehler
- [ ] Unit-Test: Schema-Validierung -- Out-of-Range-Werte (z.B. confidence > 1.0) -> Fehler
- [ ] Unit-Test: Plausibility -- EXIT + HOLD_RUNNERS gleichzeitig -> Fehler
- [ ] Unit-Test: Consistency -- Flip-Flop-Detection (zu extremer Wechsel gegenueber letztem Call)
- [ ] Unit-Test: Freshness -- Analyse abgelaufen (> 180s) -> wird als veraltet markiert
- [ ] Unit-Test: Kadenz -- Normal alle 3 Bars, HIGH_VOLATILITY jeden Bar
- [ ] Unit-Test: CircuitBreaker -- 3 aufeinanderfolgende Failures -> QUANT_ONLY Fallback
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer HTTP-Clients (Claude SDK, OpenAI) -- keine echten API-Calls in Unit-Tests

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: PromptBuilder + LlmResponseParser + SchemaValidator zusammengeschaltet (reale Klassen)
- [ ] Integrationstest: LlmAnalystOrchestrator mit MockLlmProvider -- Kadenz-Logik End-to-End
- [ ] Integrationstest: CircuitBreaker oeffnet nach 3 Failures und schliesst nach Cooldown
- [ ] Mindestens 1 Integrationstest der den vollstaendigen Pfad Tick-Daten -> Prompt -> Response -> LlmTacticalOutput durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff. LLM-Konfiguration und Cache sind in-memory.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: PromptBuilder-Code, LlmResponseParser, SchemaValidator, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei Netzwerk-Timeout (nicht HTTP-Error)? Was wenn das LLM valide JSON liefert aber mit unerwarteten Extra-Feldern? Wie wird Prompt-Versionskompatibilitaet gehandhabt wenn das Schema sich aendert?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety bei LLM-Responses, CircuitBreaker-Logik, Thread-Safety bei Fallback-Cache, Ressource-Leaks bei HTTP-Verbindungen"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/04-llm-integration.md` (alle relevanten Abschnitte) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche 3-Tier-Prompt-Aufbau, 5-Layer-Validierung, Freshness-Regeln, Kadenz-Logik, Anti-Drift-Guardrails"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Kosten-Kontrolle (zu viele LLM-Calls in HIGH_VOLATILITY)? Prompt-Laenge-Limits bei Claude/OpenAI? Was wenn beide Provider gleichzeitig ausfallen?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Fallback-Strategie, Consistency-Check-Algorithmus, Kadenz-Implementierung)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Dies ist die groesste Einzel-Story. Kann ggf. in Sub-Stories gesplittet werden (Prompt + Parser + Validation + Kadenz)
- KRITISCH: Das LLM darf KEINE Preise, Stueckzahlen oder Order-Typen liefern. Nur bounded Enums!
- Der PromptBuilder muss so gebaut sein, dass Prompt-Aenderungen versioniert werden (promptVersion Feld)
- Claude SDK verwendet "tool_use" fuer structured output -- das muss der Provider-Client korrekt nutzen
- OpenAI nutzt "function_calling" -- anderer Mechanismus, gleiche Semantik
- Fallback-Cache: Thread-Safety beachten (AtomicReference oder synchronized block, kein Lombok)
- QUANT_ONLY Fallback (DegradationMode): Wenn CircuitBreaker offen ist, arbeitet das System ohne LLM weiter
