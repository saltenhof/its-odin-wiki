# Protokoll: ODIN-005 -- Monitor-Event-Types im API-Modul

## Working State
- [x] Story + Konzepte gelesen
- [x] Bestehenden Code analysiert
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (16 Tests)
- [x] Kompilierung erfolgreich
- [x] IntegrationTest implementiert (4 Tests, Failsafe)
- [x] ChatGPT-Sparring durchgefuehrt
- [x] Gemini-Review (3 Dimensionen)
- [x] Review-Findings eingearbeitet
- [x] mvn verify PASS (83 Unit + 30 IT)
- [x] Commit & Push

## Design-Entscheidungen

### 1. Event-Type-Namen aus Konzept 01, nicht aus Story-AC
Story-AC enthielt VOLUME_SPIKE, PRICE_BREAK_LEVEL etc. Konzept (01-data-pipeline.md, Abschnitt 9)
verwendet CRASH_DOWN, SPIKE_UP etc. Konzeptnamen als autoritativ gewaehlt.
Abweichung begruendet: Story-AC-Namen sind generische Platzhalter, Konzeptnamen sind fachlich praezise.
Open Point fuer Stakeholder: Story-AC und Konzept sollten angeglichen werden (Dokumentationsfehler in Story).

### 2. Neun Event-Typen mit Default-Escalation
Jeder MonitorEventType hat eine defaultEscalation(), die aber vom Caller ueberschrieben werden kann.
Das MonitorEvent-Record traegt die tatsaechliche Escalation, die vom Default abweichen kann.

### 3. DQ_STALE -> KILL_SWITCH kein Widerspruch
Gemini hatte einen Widerspruch gesehen: Monitor "MUST NOT execute final trading actions" aber
DQ_STALE -> KILL_SWITCH. Klaerung: Der Monitor ERKENNT und ESKALIERT (publiziert MonitorEvent).
Der LifecycleManager (odin-core) EMPFAENGT das Event und AKTIVIERT den Kill-Switch.
Die Verantwortungstrennung ist: detection -> escalation event -> execution.
In JavaDoc und EscalationAction-Enum-JavaDoc explizit dokumentiert (Runde 2).

### 4. Vier Escalation-Stufen
LOG_ONLY, TRIGGER_LLM_CALL, ALERT, KILL_SWITCH. Ordinals implizieren aufsteigende Schwere.

### 5. runId nullable (Entscheidung Runde 2)
runId ist nullable (`String runId` ohne @NotNull). Begruendung: MonitorEvents koennen theoretisch
auch ausserhalb eines aktiven TradingRuns emittiert werden (z.B. waehrend System-Initialisierung
oder Warm-up-Phase). Ein null-runId ist ein Signal, dass kein RunContext aktiv ist.
Downstream-Consumer muessen null pruefen. Alternativ-Ansatz (non-null, aber "SYSTEM"-Sentinel)
wurde verworfen als over-engineered fuer diesen Scope.

### 6. MonitorEventListener-Port (Entscheidung Runde 2)
Der Port MonitorEventListener wird NICHT in ODIN-005 erstellt. Begruendung: Der Port gehoert
funktional zu ODIN-007 (Detektor-Implementierung in odin-data), das auch den Publisher/Consumer
definiert. Ein leerer Port ohne Implementierung ist kein Wert in odin-api allein.
ODIN-007 erstellt: Listener-Interface in odin-api + Implementierung in odin-data.

### 7. Jakarta Validation Annotations (Runde 2, nach Reviews)
MonitorEvent erhaelt @NotNull fuer type, marketTime, escalation sowie @NotBlank fuer instrumentId.
Die jakarta.validation-api ist als provided-Dependency bereits im Modul. runId bleibt unannotiert
(nullable per Entscheidung 5). value/threshold bleiben primitive double (0.0 als Null-Semantik
fuer Events ohne numerische Schwellenwerte ist akzeptabel und einfacher als Double nullable).

## Dateien

### Neu erstellt
- `odin-api/src/main/java/de/its/odin/api/model/EscalationAction.java` -- 4-Wert-Enum
- `odin-api/src/main/java/de/its/odin/api/model/MonitorEventType.java` -- 9-Wert-Enum mit defaultEscalation()
- `odin-api/src/main/java/de/its/odin/api/event/MonitorEvent.java` -- Event-Record mit Jakarta Validation
- `odin-api/src/test/java/de/its/odin/api/model/MonitorEventTypeTest.java` -- 16 Unit-Tests (Surefire)
- `odin-api/src/test/java/de/its/odin/api/monitor/MonitorEventIntegrationTest.java` -- 4 Integration-Tests (Failsafe)

## ChatGPT-Sparring

**Datum:** 2026-02-21
**Dateien gesendet:** MonitorEventType.java, EscalationAction.java, MonitorEvent.java, MonitorEventIntegrationTest.java

### ChatGPT-Feedback (Zusammenfassung)

ChatGPT hat strukturiertes Feedback in 3 Bereichen gegeben:

**1. API-Design und Immutabilitaet:**
- MonitorEvent fehlen Validation-Annotations (P0): @NotNull, @NotBlank fehlten
- value/threshold als primitive double: bei booleschen Events (VWAP_CROSS, EMA_CROSS) entsteht 0.0-Magie
- KILL_SWITCH JavaDoc klingt wie direkte Aktion, nicht Eskalation -- unklar ob Consumer verwirrt werden

**2. Trading-System-Vollstaendigkeit:**
- Kein eventId/dedupe-Key fuer Idempotenz bei Retries/Replay
- Kein detectedAt (System-Zeitstempel) neben marketTime fuer Latenz-Monitoring
- LLM-Throttling fehlt: CRASH_DOWN kann minutenlang spammen -- mindestens in JavaDoc erwaehnen
- Severity bei ALERT undifferenziert (WARNING vs CRITICAL)

**3. JavaDoc/Validierung:**
- instrumentId: unklar ob Ticker, FIGI oder IBKR-ConId
- value/threshold Semantik je Eventtyp nicht dokumentiert (Einheit, Vergleichsrichtung)
- VWAP/EMA Cross: haeufig, Debouncing sollte in JavaDoc erwaehnt werden

### Bewertung und adressierte Findings

**Adressiert:**
- @NotNull/@NotBlank Annotations auf MonitorEvent hinzugefuegt (P0)
- EscalationAction.KILL_SWITCH JavaDoc korrigiert: "signal to risk controller", nicht direkte Aktion
- MonitorEventType JavaDoc: exakt die 3 erlaubten Eskalationen aufgelistet, Debouncing-Hinweis fuer VWAP/EMA ergaenzt
- MonitorEvent JavaDoc: instrumentId als "ticker / internal instrument identifier" praezisiert, value/threshold Semantik nach Event-Typ erklaert

**Verworfen (begruendet):**
- eventId: out of scope fuer ODIN-005, gehoert zu ODIN-007 Scope (Event-Routing)
- detectedAt: Consumer-Concern, nicht API-Schicht. Downstream kann Empfangszeit selbst stempeln
- Severity-Enum fuer ALERT: over-engineered fuer S-Story. Kann in ODIN-007 ergaenzt werden wenn noetig
- Double nullable statt primitive double: 0.0-Konvention ist einfacher und ausreichend dokumentiert

## Gemini-Review

**Datum:** 2026-02-21
**Dateien gesendet:** MonitorEventType.java, EscalationAction.java, MonitorEvent.java, MonitorEventIntegrationTest.java

### Dimension 1: Code-Bugs und technische Korrektheit

Gemini fand keine kritischen Laufzeit-Bugs. Der Code ist modern (Records), sauber dokumentiert.
Gefundene Luecken:
- Fehlende Jakarta Validation Annotations (obwohl jakarta.validation verfuegbar ist)
- Primitive double fuer value/threshold: 0.0 als Null-Semantik fuer boolean-Events (VWAP_CROSS) ist implizit
- Test-Architektur technisch korrekt, aber "IntegrationTest" testet eher Konstruktor-/Value-Semantics als End-to-End

**Adressiert:** @NotNull/@NotBlank auf MonitorEvent ergaenzt. value/threshold Semantik im JavaDoc erklaert.
Double-Primitives behalten (0.0-Konvention explizit dokumentiert als akzeptable Entscheidung).

### Dimension 2: Konzepttreue zum Trading-System

Gemini hat die Konzepttreue als stark eingestuft:
- Separation of Concerns (detection -> escalation -> execution) klar modelliert
- DQ_STALE -> KILL_SWITCH korrekt (Eskalation, nicht direkte Aktion)
- TRIGGER_LLM_CALL auf CRASH_DOWN/SPIKE_UP korrekt fuer Intraday-LLM-Refresh zwischen 3m-Zyklen
- Risk-First-Denken bei DQ_STALE professionell bewertet

**Abweichung:** Kein Befund -- alle 9 Eventtypen und Eskalationszuordnungen sind konzeptkonform.

### Dimension 3: Praxis-Gaps

Gemini identifizierte 4 relevante Praxis-Luecken:
1. **LLM-Spam-Schutz:** CRASH_DOWN kann minutenlang triggern -- Throttling/Debouncing fehlt
2. **System Time vs. Market Time:** detectedAt fehlt fuer Latenz-Monitoring
3. **Kill-Switch Granularitaet:** DQ_STALE auf NVDA -> System-weiter Kill-Switch vs. Instrument-Level?
4. **Kontext-Payload:** EXHAUSTION_CLIMAX braucht mehr als ein double fuer forensische Analyse

**Adressiert:**
- Debouncing-Hinweis fuer VWAP/EMA in MonitorEventType JavaDoc ergaenzt
- Kill-Switch Granularitaet als Open Point dokumentiert (architektonische Entscheidung fuer odin-core)

**Verworfen (begruendet):**
- detectedAt: Consumer-Responsibility, nicht API-Schicht
- contextData/Map-Payload: zu komplex fuer ODIN-005 Scope; kann in ODIN-007 evaluiert werden

## Offene Punkte

### OP-1: Story-AC vs. Konzept-Namensdivergenz (Minor)
Die Story-AC nennt VOLUME_SPIKE, PRICE_BREAK_LEVEL etc. -- das Konzept verwendet andere Namen.
Stakeholder sollte Story-AC und Konzept angleichen. Kein Code-Defekt, Dokumentationsfehler in Story.
**Entscheidung:** Konzeptnamen sind autoritativ. Story-AC ist zu korrigieren.

### OP-2: runId nullable (Entschieden)
runId ist nullable. Begruendung und Entscheidung unter Design-Entscheidung 5 dokumentiert.

### OP-3: MonitorEventListener-Port (Entschieden)
Port wird in ODIN-007 erstellt, nicht in ODIN-005. Begruendung unter Design-Entscheidung 6 dokumentiert.

### OP-4: Kill-Switch Granularitaet (Offen, Architekturentscheidung odin-core)
Gemini hat korrekt bemerkt: DQ_STALE auf einem einzelnen Instrument loest aktuell einen System-weiten
Kill-Switch aus (DAY_STOPPED). Frage: Sollte es ein Instrument-Level-Kill-Switch geben (Instrument
pausieren, andere laufen lassen)?
**Status:** Offen. Architekturentscheidung gehoert zu odin-core/LifecycleManager-Design.
Empfehlung: In ODIN-007 oder als eigenes ADR adressieren.

### OP-5: LLM-Throttling fuer hochfrequente Events (Offen, ODIN-007)
CRASH_DOWN/SPIKE_UP koennen bei starker Volatilitaet mehrfach pro Minute feuern.
Ein TRIGGER_LLM_CALL pro Minute ist dann zu viel. Throttling/Debouncing ist Consumer-seitig
(im Detector oder im LLM-Call-Handler). Wird in ODIN-007 beruecksichtigt.
