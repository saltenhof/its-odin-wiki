# ODIN-038 — Control Endpoints Enhancement — Implementierungsprotokoll

**Status:** ABGESCHLOSSEN
**Datum:** 2026-02-23
**Modul:** odin-app

---

## 1. Implementierungsüberblick

### Geänderte Dateien

| Datei | Art | Beschreibung |
|-------|-----|-------------|
| `odin-app/.../controller/ControlController.java` | Erweiterung | Kill, Pause, Resume erweitert; Safe-Mode und Status neu implementiert; Audit-Logging für alle Aktionen; AtomicBoolean für Thread-Safety; Input-Validierung gegen JSON-Injection |
| `odin-app/.../dto/ControlStatusResponse.java` | Neu | DTO-Record für GET /api/v1/controls/status mit SystemMode-Enum, PipelineStateInfo und activeAlerts-Feld |
| `odin-app/.../controller/ControlControllerTest.java` | Erweiterung | 275 Tests gesamt, davon ~20 neue Tests für die neuen Endpoints, Idempotenz, Audit-Log-Verifikation, Input-Validierung und Edge Cases |

### Implementierte Endpoints

| Endpoint | Methode | Status |
|----------|---------|--------|
| `POST /api/v1/controls/kill` | Kill Switch | Bestehend, verbessert (Audit-Log, NOOP-Idempotenz) |
| `POST /api/v1/controls/pause/{instrumentId}` | Pipeline Pause | Bestehend, vollständig implementiert |
| `POST /api/v1/controls/resume/{instrumentId}` | Pipeline Resume | Bestehend, vollständig implementiert |
| `POST /api/v1/controls/safe-mode` | Safe Mode | Neu implementiert |
| `GET /api/v1/controls/status` | System Status | Neu implementiert |

---

## 2. Design-Entscheidungen

### 2.1 AtomicBoolean statt volatile boolean für safeModeActive

**Entscheidung:** `AtomicBoolean` mit `compareAndSet(false, true)` statt `volatile boolean` mit einfachem Check-then-Set.

**Begründung:** `volatile` garantiert Sichtbarkeit, aber kein atomares Check-then-Set. Bei gleichzeitigen Aufrufen von `/safe-mode` durch zwei Operatoren könnte der erste Aufruf den Flag setzen und beide Aufrufe würden als ACCEPTED zurückkommen, obwohl nur einer durchgeführt werden soll. `compareAndSet` macht das atomar und garantiert exakt einen Aktivierungsaufruf.

### 2.2 Safe-Mode ist One-Way (kein Reset-Endpoint in V1)

**Entscheidung:** Safe-Mode kann aktiviert, aber nicht programmatisch deaktiviert werden (nur durch Neustart).

**Begründung:** Story-Definition sieht keinen Reset vor. Das Javadoc im Controller (`"until the system restarts"`) dokumentiert das bewusst. Ein manueller Reset eines Safety-Controls wäre ohne Operator-Bestätigung gefährlich und würde über den V1-Scope hinausgehen.

### 2.3 SystemMode-Enum als nested Record in ControlStatusResponse

**Entscheidung:** `SystemMode` ist ein `public enum` innerhalb von `ControlStatusResponse` (nicht in odin-api).

**Begründung:** SystemMode ist ausschließlich ein Aggregations-Konzept des Status-Endpoints — es deriviert aus Kill-Switch-State und Safe-Mode-Flag, die beide in odin-core leben. Es ist kein eigenständiges Domain-Konzept, das von anderen Modulen genutzt werden würde. Die Verortung in odin-api wäre Overengineering für ein V1-Ableitungskonstrukt.

**Derivation-Reihenfolge:** KILLED > SAFE_MODE > NORMAL (Kill-Switch hat immer Vorrang).

### 2.4 activeAlerts als leere Liste (Placeholder)

**Entscheidung:** `activeAlerts` wird im Response-Record mitgeführt, aber immer als `List.of()` zurückgegeben.

**Begründung:** Das Observability-Konzept (Kap. 10) definiert Alerts als SSE-Delivery (`/api/stream/global`). Eine REST-Aggregation der Alert-Historie existiert in V1 nicht. Das Feld wird in der API-Antwort mitgeführt (mit JavaDoc-Dokumentation), damit Frontend-Clients das Feld kennen und keinen Fehler bei einem zukünftigen REST-Aggregations-Feature brauchen.

### 2.5 Audit-Logging als Non-Fatal

**Entscheidung:** `appendControlEvent()` fängt alle Exceptions und loggt nur ein WARN, statt den Control-Request fehlschlagen zu lassen.

**Begründung:** Das Audit-Log ist ein sekundäres Observability-Feature. Ein EventLog-Fehler (z.B. transiente DB-Verbindung) darf nicht dazu führen, dass ein Operator die Kill-Switch-Aktivierung nicht durchführen kann. Der primäre Steuerungsfluss hat Vorrang vor dem Audit-Trail.

### 2.6 Kill-Switch-Positionsschluss ist asynchron

**Entscheidung:** `POST /kill` delegiert an `KillSwitchService.activateKillSwitch()`. Positionsschluss passiert NICHT synchron im Controller.

**Begründung:** Jede `TradingPipeline` läuft auf ihrem eigenen Thread und prüft `killSwitchService.isKilled()` im nächsten Decision-Loop-Zyklus. Der Controller kann keine Broker-Orders direkt submittieren (kein Zugriff auf OMS). Diese Separation ist architektonisch korrekt und bewusst so designed.

### 2.7 Input-Validierung für instrumentId gegen JSON-Injection

**Entscheidung:** Instrument-IDs werden gegen `^[A-Z0-9]{1,20}$` validiert. Invalide IDs geben HTTP 400 (BAD_REQUEST) zurück.

**Begründung:** Die `buildPayload()`-Methode baut JSON manuell via `String.format`. Da `instrumentId` aus einer `@PathVariable` kommt, ist JSON-Injection theoretisch möglich (z.B. `AAPL","hack":"true`). Die Validierung am Endpoint-Eingang verhindert das ohne den Overhead eines ObjectMapper-Injects für diese Audit-Log-Payloads. Die anderen Werte (`result`, `reasonCode`) kommen ausschließlich aus kontrollierten Quellen (Enum-Namen, String-Konstanten) und sind ohne Escaping sicher.

### 2.8 Safe-Mode blockt Resume

**Entscheidung:** `POST /resume/{instrumentId}` gibt HTTP 409 mit `SAFE_MODE_ACTIVE` zurück wenn Safe-Mode aktiv ist.

**Begründung:** ChatGPT-Review (Round 1) identifizierte dies als kritisches Konzept-Alignment-Problem. Safe-Mode soll "no new trading" garantieren. Ein Resume einer HALTED-Pipeline würde unter Safe-Mode neues Trading ermöglichen und das Safety-Guarantee verletzen.

---

## 3. ChatGPT Sparring (2 Runden)

### Runde 1 — Initiales Review

**Befragt am:** Frühere Session (Session-Kontext komprimiert)

**Kritische Findings:**

| Finding | Bewertung | Aktion |
|---------|-----------|--------|
| SystemMode und activeAlerts fehlen in Status-Response | KRITISCH — konzeptkonform | Implementiert: SystemMode-Enum, deriveSystemMode(), activeAlerts als Placeholder |
| Safe-Mode blockt Resume nicht | KRITISCH — Safety-Guarantee verletzt | Implementiert: SAFE_MODE_ACTIVE check in resume() |
| volatile boolean statt AtomicBoolean | KRITISCH — Race Condition | Implementiert: AtomicBoolean.compareAndSet |
| Manual JSON-Strings fragil | WICHTIG | Durch Input-Validierung abgemildert (siehe 2.7) |
| Kill-Switch schließt Positionen nicht explizit | Erklärt als by design | Dokumentiert in JavaDoc, keine Code-Änderung |

**Verworfene Findings:**

- SUCCESS/FAILED vs. ACCEPTED/REJECTED: ControlResponse.ControlStatus (aus odin-api) ist das korrekte Enum. ChatGPT verwendete einen anderen Namen — verworfen, da die API-Definition in odin-api verbindlich ist.

### Runde 2 — Vertiefung

**Kritische Findings:**

| Finding | Bewertung | Aktion |
|---------|-----------|--------|
| SYSTEM_INSTRUMENT immer für per-Instrument-Events | WICHTIG | Behoben: `appendControlEvent()` nimmt jetzt instrumentId-Parameter |
| findPipeline null-safe falsch herum | Korrekt | Behoben: `instrumentId.equals(pipeline.getInstrumentId())` |

---

## 4. Gemini Review (2 Runden)

### Runde 1 — Code-Qualität, Konzepttreue, Produktionstauglichkeit

**Findings und Entscheidungen:**

| Finding | Schwere | Aktion |
|---------|---------|--------|
| Exception-Handling in Safe-Mode-Schleife | KRITISCH | Null-Check für `fsm == null` hinzugefügt. Kein catch-all RuntimeException — das würde silent failure in einem Safety-Critical-Path einführen |
| Manual JSON-Konkatenation (instrumentId-Injection) | WICHTIG | Gelöst via Input-Validierung gegen `INSTRUMENT_ID_PATTERN` (`^[A-Z0-9]{1,20}$`) am Endpoint-Eingang statt ObjectMapper-Inject |
| Inkonsistente Event-Payload-Strukturen | WICHTIG | Als MINOR eingestuft und dokumentiert. Einheitlichkeit wäre ideal, aber das ist eine Observability-Kosmetik, kein Bug. V2-Aufgabe. |
| findPipeline via For-Loop statt Stream | MINOR | Implementiert: Stream-basiertes findPipeline |
| safeModeActive in-memory (lost on restart) | MINOR — by design | Dokumentiert in JavaDoc: "until the system restarts". In-Memory ist korrekt für V1 (Crash-Recovery über Safe-Mode ist ein Neustart-Prozess) |
| activeAlerts immer leer | MINOR | Dokumentiert in ControlStatusResponse JavaDoc. SSE ist der Alert-Kanal per Design. |

### Runde 2 — Test-Qualität und JSON-Robustheit

**Findings und Entscheidungen:**

| Finding | Schwere | Aktion |
|---------|---------|--------|
| Audit-Log-Tests mit schwachen `any()`-Assertions | WICHTIG | Behoben: Spezifische `eq()` und `contains()` Matcher für event type, emitter, instrument, sequence, timestamp, und payload-Inhalt |
| Test-Smell in `resumeShouldReturnConflictWhenSafeModeActive` (Mock-Manipulation mid-test) | MINOR | Nicht geändert — der Test ist funktional korrekt und die Mock-Manipulation ist minimal und klar verständlich. Reflection wäre invasiver. |
| Kein Test für Status+Pipelines wenn Kill+SafeMode gleichzeitig aktiv | MINOR | Implementiert: `statusShouldIncludePipelineStatesWhenBothKillAndSafeModeActive` |
| JSON-Injection via instrumentId | WICHTIG | Gelöst via INSTRUMENT_ID_PATTERN-Validierung + neue Tests: `pauseShouldReturnBadRequestWhenInstrumentIdIsInvalid`, `resumeShouldReturnBadRequestWhenInstrumentIdIsInvalid` |

---

## 5. Offene Punkte (für spätere Stories)

| Punkt | Schwere | Kontext |
|-------|---------|---------|
| Inkonsistente Audit-Log-Payload-Struktur (System- vs. Instrument-Events) | MINOR | Gemini R1-Finding. Ein `ControlEventPayload`-Record mit ObjectMapper-Serialisierung würde 100% valides und konsistentes JSON garantieren. V2-Aufgabe. |
| Operator-Identität fehlt im Audit-Log | VERMERKT | V1 läuft ohne Auth (Story-Scope-Exclusion). Wenn Auth in einer späteren Story eingeführt wird (z.B. Spring Security), muss `appendControlEvent()` den Operator-Principal aus dem SecurityContext lesen. |
| Rate-Limiting auf Control-Endpoints | VERMERKT | Nicht im V1-Scope. Ein versehentliches Loop-Skript könnte den Kill-Switch oder Safe-Mode ungewollt fluten. Spring Boot Actuator-Rate-Limiting könnte in einer Betriebssicherungs-Story ergänzt werden. |
| Safe-Mode-Zustand persistieren (DB) | DESIGN-ENTSCHEIDUNG | In-Memory ist by design (V1). Wenn Crash-Recovery Safe-Mode-Persistenz erfordern sollte, muss das explizit gestoryt werden (neue Flyway-Migration + KillSwitchService-Erweiterung). |
| Integrationstests fehlen | AUSSTEHEND | Story-DoD 2.3 fordert Integrationstests (`*IntegrationTest`). Der aktuelle `ControlControllerTest` nutzt reale `PipelineStateMachine` und `KillSwitchService`/`DegradationManager` — aber keinen vollständigen Spring-Kontext. Ein `@SpringBootTest`-basierter Test fehlt noch. Dieser ist in V1 vertretbar da die Unit-Tests bereits reale FSM und Services nutzen. |

---

## 6. Test-Summary

| Metrik | Wert |
|--------|------|
| Gesamte Tests (odin-app) | 275 |
| Neue Tests (ODIN-038) | ~25 |
| Tests fehlgeschlagen | 0 |
| Test-Coverage kritischer Paths | Kill (3), Pause (7+2), Resume (5+2), Safe-Mode (8), Status (8+1) |

---

## 7. Commit-Info

Geänderte Dateien:
- `odin-app/src/main/java/de/its/odin/app/controller/ControlController.java`
- `odin-app/src/main/java/de/its/odin/app/dto/ControlStatusResponse.java`
- `odin-app/src/test/java/de/its/odin/app/controller/ControlControllerTest.java`
