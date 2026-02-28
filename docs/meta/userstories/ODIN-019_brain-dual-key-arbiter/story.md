# ODIN-019: Dual-Key Arbiter (Symmetric Hybrid Protocol)

**Modul:** odin-brain
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-011 (Gate Cascade), ODIN-002 (LLM Tactical Schema), ODIN-017 (LLM Tactical Controller)

---

## Kontext

Der aktuelle `DecisionArbiter` ist einseitig: Rules liefern Intent, Quant validiert. Das Konzept definiert ein Symmetric Hybrid Protocol mit Dual-Key-Entscheidungen: QuantVote und LlmVote muessen BEIDE zustimmen (bei Entries). Der Arbiter fused die Votes und bestimmt die finale Aktion. Die Arbiter-Logik ist deterministisch und laedt keine eigene Verantwortung.

## Scope

**In Scope:**
- Fundamentaler Umbau von `DecisionArbiter` zum Dual-Key-Modell
- QuantVote: Ergebnis der Gate-Cascade + Rules (deterministic)
- LlmVote: Ergebnis der LLM Tactical Analysis (bounded)
- Fusion-Regeln fuer Entry, Exit, und Hold-Entscheidungen
- Parameter-Fusion: Bei Einigung auf Aktion, wie werden die Parameter (Stop, Target, Trail) bestimmt?
- Konservativitaets-Prinzip: Bei Widerspruch gewinnt die konservativere Seite
- QUANT_ONLY Fallback (DegradationMode): Arbiter arbeitet ohne LlmVote wenn LLM nicht verfuegbar

**Out of Scope:**
- Aenderungen an GateCascadeEvaluator oder RulesEngine (ODIN-011 ist Vorbedingung)
- Aenderungen am LLM-Call-Mechanismus (ODIN-017 ist Vorbedingung)
- Frontend-Visualisierung der Fusion-Entscheidungen

## Akzeptanzkriterien

- [ ] Entry: QuantVote=ENTER + LlmVote=ENTER -> ENTER (beide muessen zustimmen)
- [ ] Entry: QuantVote=ENTER + LlmVote=NO_TRADE -> NO_ACTION (LLM-Veto)
- [ ] Entry: QuantVote=NO_GATE + LlmVote=ENTER -> NO_ACTION (Quant-Veto via Gate-Fail)
- [ ] Exit: QuantVote=EXIT ODER LlmVote=EXIT -> EXIT (einer reicht)
- [ ] Hold: Wenn Position vorhanden und kein Exit-Signal -> HOLD
- [ ] Parameter-Fusion: Trail-Breite = konservativerer Wert (engerer Trail gewinnt fuer Profit-Sicherung)
- [ ] Parameter-Fusion: Target-Policy folgt LLM (wenn plausibel)
- [ ] Parameter-Fusion: Profit-Protection-Profile folgt LLM (wenn plausibel)
- [ ] Regime-Fusion: KPI-Regime primaer, LLM-Regime sekundaer, Konservativitaets-Rangfolge
- [ ] Alle Fusion-Entscheidungen werden im EventLog dokumentiert (welche Seite "gewann")
- [ ] QUANT_ONLY Modus: Arbiter arbeitet nur mit QuantVote, LlmVote wird ignoriert

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/DecisionArbiter.java` (fundamentaler Umbau)
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/QuantVote.java` (neues Record)
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/LlmVote.java` (neues Record)
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/FusionResult.java` (neues Record)

**Regime-Konservativitaets-Rangfolge:** UNCERTAIN > TREND_DOWN > RANGE_BOUND > HIGH_VOLATILITY > TREND_UP (konservativster Regime gewinnt bei Widerspruch).

## Konzept-Referenzen

- `docs/concept/08-symmetric-hybrid-protocol.md` -- Abschnitt 1 "Dual-Key-Entscheidungsprinzip"
- `docs/concept/08-symmetric-hybrid-protocol.md` -- Abschnitt 2 "QuantVote und LlmVote Schemas"
- `docs/concept/08-symmetric-hybrid-protocol.md` -- Abschnitt 3 "Arbiter-Logik"
- `docs/concept/08-symmetric-hybrid-protocol.md` -- Abschnitt 4 "Parameter-Fusion-Regeln"
- `docs/concept/08-symmetric-hybrid-protocol.md` -- Abschnitt 5 "Regime-Fusion"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- "ODIN ist KEIN reines Algo-Trading-System. LLM MUSS Entscheidungsgewalt haben"

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (QuantVote, LlmVote, FusionResult als Java Records)
- [ ] ENUM statt String fuer endliche Mengen (VoteAction, FusionAction, DegradationMode)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.arbiter.*`
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Entry -- alle 4 Entry-Kombinationen (ENTER+ENTER, ENTER+NO_TRADE, NO_GATE+ENTER, NO_GATE+NO_TRADE)
- [ ] Unit-Test: Exit -- QuantVote=EXIT (LlmVote egal) -> EXIT; LlmVote=EXIT (QuantVote egal) -> EXIT
- [ ] Unit-Test: Hold -- keine Exit-Signale + Position vorhanden -> HOLD
- [ ] Unit-Test: Parameter-Fusion -- engerer Trail gewinnt (Konservativitaet)
- [ ] Unit-Test: Regime-Fusion -- Konservativitaets-Rangfolge (UNCERTAIN schlaegt TREND_UP)
- [ ] Unit-Test: EventLog-Eintraege -- Fusion-Entscheidung wird geloggt mit Begruendung
- [ ] Unit-Test: QUANT_ONLY Fallback -- LlmVote nicht verfuegbar -> Arbiter entscheidet nur auf Basis QuantVote
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer GateCascadeEvaluator, LlmAnalystOrchestrator, EventLog (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `DecisionArbiter` mit realem `GateCascadeEvaluator` (reale Klassen zusammengeschaltet)
- [ ] Integrationstest: Vollstaendiger Entscheidungspfad -- Tick-Daten -> QuantVote + LlmVote -> FusionResult -> Aktion
- [ ] Integrationstest: QUANT_ONLY Degradation -- CircuitBreaker offen -> Arbiter laeuft trotzdem fehlerfrei
- [ ] Mindestens 1 Integrationstest der bestaetigt, dass ohne LLM-Zustimmung kein Entry stattfindet
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- der Arbiter hat keinen direkten Datenbankzugriff. EventLog-Eintraege ueber Port-Interface.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: DecisionArbiter-Code, QuantVote/LlmVote/FusionResult-Records, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei simultanen Exit-Signalen von beiden Seiten? Was wenn Regime-Fusion zu UNCERTAIN fuehrt aber QuantVote ENTER ist? Wie wird mit teilweiser LlmVote (nur manche Felder vorhanden) umgegangen?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, fehlerhafte Fusion-Logik, Null-Safety bei LlmVote-Feldern, Konservativitaets-Rangfolge-Implementierung"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/08-symmetric-hybrid-protocol.md` (alle Abschnitte) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche Fusion-Regeln fuer Entry/Exit/Hold, Parameter-Fusion, Regime-Fusion, QUANT_ONLY Fallback, EventLog-Format"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Was passiert bei Multi-Tranche-Positionen -- wird jede Tranche separat votiert oder die Gesamtposition? Was wenn QuantVote und LlmVote sehr unterschiedliche Trail-Breiten vorschlagen -- ist 'engerer gewinnt' immer richtig?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Fusion-Algorithmus, QUANT_ONLY-Trigger-Bedingung, Regime-Rangfolge-Implementierung)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Die Fusion-Logik in Konzept 08 Abschnitt 3 ist die normative Referenz
- QUANT_ONLY Modus (DegradationMode): Arbiter arbeitet nur mit QuantVote, LlmVote wird ignoriert
- Regime-Fusion Konservativitaets-Rangfolge: UNCERTAIN > TREND_DOWN > RANGE_BOUND > HIGH_VOLATILITY > TREND_UP
- Die JSON-Schema-Beispiele in Konzept 08 Abschnitt 6 zeigen das erwartete Format
- Abhaengigkeit zu ODIN-011: GateCascadeEvaluator muss vorhanden sein
- Abhaengigkeit zu ODIN-017: LlmTacticalOutput-Schema muss definiert sein (LlmVote basiert darauf)
