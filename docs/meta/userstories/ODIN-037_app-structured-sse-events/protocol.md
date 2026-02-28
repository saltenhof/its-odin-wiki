# ODIN-037 Protocol — Structured SSE Events (Monitoring Streams)

**Story:** ODIN-037
**Implementiert:** 2026-02-23
**Status:** DONE — 253 Unit-Tests gruen, bereit fuer QS-Agent/Commit

---

## 1. Implementation Summary

### Was wurde implementiert

**SseEventType (odin-app/src/main/java/de/its/odin/app/sse/SseEventType.java)**
- Von Utility-Klasse mit `static final String`-Konstanten zu proper ENUM konvertiert
- `.value()` Methode liefert den kebab-case SSE-Event-Namen
- Alle 8 Monitoring-Types + HEARTBEAT + 4 Backtest-Types

**8 Event-DTO Records (odin-app/src/main/java/de/its/odin/app/dto/sse/)**
- `PipelineStateEvent` — FSM-Zustandsuebergaenge
- `PositionUpdateEvent` — Position-Snapshot (quantity, avgEntryPrice nullable wenn flat, unrealizedPnl, realizedPnl)
- `IndicatorUpdateEvent` — KPI-Snapshot (ema50, ema100, rsi, atr, regime, subregime)
- `LlmAnalysisEvent` — LLM-Entscheidung (action, confidence, reasoning truncated to 200 chars)
- `TradeExecutedEvent` — Fill-Bestaetigung (side, quantity, price, reason)
- `AlertEvent` — System-Alerts (AlertLevel: INFO/WARNING/CRITICAL/EMERGENCY, message, timestamp)
- `DegradationChangeEvent` — Degradierungs-Modus-Wechsel (oldMode, newMode, reason)
- `CycleUpdateEvent` — Neuer Trading-Cycle (cycleNumber, cycleStartTime, previousCyclePnl nullable)

**StreamController (odin-app/src/main/java/de/its/odin/app/controller/StreamController.java)**
- Endpunkt umgeschrieben: `pipelineStream` an `/pipeline/{instrumentId}` (ohne Timeframe-Parameter)
- `last-event-id` Header-Parsing mit fallback auf 0 bei ungueltigem Wert
- Global- und Backtest-Streams unveraendert

**SseEmitterManager (odin-app/src/main/java/de/its/odin/app/sse/SseEmitterManager.java)**
- Memory-Leak-Fix: `addEmitter`/`removeEmitter` private Methoden mit Stream-Key-Cleanup
- Wenn letzter Emitter eines Streams entfernt wird: Map-Eintrag wird entfernt
- Backtest-Streams (UUID-Keys mit `backtest-`-Prefix): RingBuffer wird bei letztem Disconnect ebenfalls bereinigt
- Replay-Fix: `catch (Exception)` statt nur `catch (IOException)` — `IllegalStateException` wird korrekt behandelt
- `@PreDestroy` auf `shutdown()` — Heartbeat-Scheduler wird bei Spring Context Shutdown sauber beendet
- Heartbeat-Cleanup und sendEvent-Cleanup verwenden beide `removeEmitter` fuer konsistentes Stream-Key-Management

### Design-Entscheidungen

| Entscheidung | Begruendung |
|---|---|
| `SseEventType` als ENUM (nicht String-Konstanten) | Typsicherheit, kein Tippfehler-Risiko, ODIN-Coding-Regel |
| Records fuer alle Event-DTOs | Immutability, kompakt, kein Boilerplate, ODIN-Coding-Regel |
| `type`-Feld in jedem DTO = SSE event name | Discriminated Union Pattern: Frontend parsed `event.data.type` als Discriminator |
| Factory-Methoden `of(...)` auf Records | Konsistenz, type-Discriminator immer korrekt gesetzt |
| `MAX_REASONING_LENGTH = 200` in LlmAnalysisEvent | SSE-Payload soll klein bleiben; detailliertes Reasoning ist Audit-Log-Angelegenheit |
| `avgEntryPrice`/`unrealizedPnl` als `Double` (nullable) | Flat-Position hat keinen Entry-Preis und kein unrealized PnL |
| Backtest-RingBuffer-Cleanup bei letztem Disconnect | UUID-Keys akkumulieren ohne Cleanup endlos (DoS-Risiko) |
| `Instant.now()` in SseRingBuffer bleibt | Akzeptabel fuer SSE-Infrastruktur (nicht Trading-Codepfad). MarketClock-Regel gilt fuer Trading-Entscheidungen, nicht fuer Infrastruktur-Timestamps |

---

## 2. Tests

### Test-Klassen und Coverage

| Testklasse | Art | Tests | Inhalt |
|---|---|---|---|
| `SseEventDtoSerializationTest` | Unit | 18 | JSON-Serialisierung aller 8 DTOs, type-Diskriminator, LLM-Truncation |
| `SseEventTypeTest` | Unit | 12 | ENUM-Werte, Uniqueness, Vollstaendigkeit der 13 Event-Types |
| `SseEmitterManagerTest` | Unit | 11 | Emitter-Registration, Count-Tracking, Stream-Key-Cleanup |
| `SseRingBufferTest` | Unit | 11 | Ring-Buffer-Overflow, Replay, Time-Expiry, Invarianten-Guards |
| `HeartbeatConfigTest` | Unit | 3 | Heartbeat-Intervall-Konfiguration |
| `SseStreamIntegrationTest` | Integration (Failsafe) | 12 | End-to-End: Routing, Buffering, Replay, Disconnect-Cleanup, Overflow |
| `StreamControllerTest` | Unit | 8 | Controller-Routing, Header-Parsing |

**Gesamt: 253 Unit-Tests, alle gruen.**

---

## 3. ChatGPT Review (2 Runden)

### Runde 1 — Findings

**KRITISCH (umgesetzt):**
1. **Stream-Key/RingBuffer Memory-Leak:** Beim Disconnect wurde der Map-Eintrag nie entfernt. Backtest-Streams mit UUID-Keys akkumulierten endlos.
   **Fix:** `addEmitter`/`removeEmitter` Methoden mit `computeIfPresent` — Key wird entfernt wenn Liste leer. Backtest-RingBuffer wird ebenfalls bereinigt.

2. **Replay nur `IOException` gefangen:** `SseEmitter#send()` kann auch `IllegalStateException` werfen (z.B. bei emitter state).
   **Fix:** `catch (Exception exception)` in `replayEvents()` + `safeComplete()` + `removeEmitter()`.

3. **`@PreDestroy` fehlte auf `shutdown()`:** Heartbeat-Scheduler-Thread blieb bis JVM-Ende laufen.
   **Fix:** `@PreDestroy` hinzugefuegt.

**WICHTIG (akzeptiert / begruendet verworfen):**
- **NaN/Infinity-Guards:** In ODIN nicht relevant — KPI-Werte werden vor Publish durch die KPI-Engine validiert. Low-Priority.
- **`instrumentId`-Validierung im Controller:** Sinnvoll, aber ausserhalb ODIN-037-Scope. Als offener Punkt erfasst.
- **Backtest-Eventtypen-Namespace:** `progress`, `completed` etc. sind sehr generisch. Fuer Multi-Stream relevant, aber in ODIN ein Backtest-Stream pro UUID. Akzeptabel.
- **`HEARTBEAT` ohne DTO-Record:** HEARTBEAT ist kein fachliches Event, sondern Keep-Alive. Pre-serialisierter String `{"type":"heartbeat"}` ist hier angemessener als ein Record.

### Runde 2 — Replay-Exception-Handling

ChatGPT empfahl `safeCompleteWithError()` mit Fallback zu `complete()`. Umgesetzt als vereinfachtes `safeComplete()` — `completeWithError()` wuerde einen Client-seitigen Error-Callback ausloesen, der zu unnoetigen Reconnect-Attempts fuehrt. `complete()` (graceful) ist in diesem Kontext besser.

**Bewertung:** Alle KRITISCH-Findings wurden umgesetzt. WICHTIG-Findings kritisch bewertet und teilweise verworfen.

---

## 4. Gemini Review

**Durchgefuehrt:** 2026-02-23 (QS-Agent, da IMPL-Agent-Pool unresponsive war)
**Slot:** `odin-037-qs-gemini`

### Dimension 1: Code-Qualitaet — LGTM

- Alle DTOs sauber als `record` — LGTM
- Kein `var`, explizite Typisierung ueberall — LGTM
- Magic Numbers vermieden (`MAX_REASONING_LENGTH = 200`, `SHUTDOWN_WAIT_SECONDS = 5L`) — LGTM
- JavaDoc vollstaendig, professionell formuliert, referenziert Konzept-Docs — LGTM
- Factory-Methoden (`of(...)`) in jedem DTO vorhanden, injizieren `type`-Discriminator korrekt — LGTM
- `SseEventType` als vorbildliches ENUM mit `value()`-Methode — LGTM

### Dimension 2: Konzepttreue (SSE Event Schema) — LGTM

- Alle 8 strukturierten Event-Typen vorhanden und korrekt abgebildet — LGTM
- Jedes DTO hat `type`-Feld als erster Parameter (Discriminated Union Pattern) — LGTM
- Truncation in `LlmAnalysisEvent` auf 200 Zeichen, null-safe — LGTM
- Nullable Felder korrekt als `Double` (Wrapper) modelliert — LGTM

### Dimension 3: Produktionstauglichkeit

- Memory-Leak-Fix: `removeEmitter` mit `computeIfPresent` + null-Return korrekt — LGTM
- Backtest-RingBuffer-Cleanup: `isBacktestStream` Guard korrekt — LGTM
- Replay-Exception-Handling: `catch (Exception)` ausreichend, `safeComplete()` vs. `completeWithError()` korrekt begruendet — LGTM
- `@PreDestroy` Heartbeat-Shutdown: `awaitTermination` + Fallback `shutdownNow()` vorbildlich — LGTM
- Daemon-Thread fuer Heartbeat: korrekt konfiguriert — LGTM
- `CopyOnWriteArrayList` fuer Emitter-Listen: ideal fuer leselastiges Observer-Pattern — LGTM
- **WICHTIG: Potenzielle Race Condition im Replay-Pfad** — `addEmitter` wird VOR `replayEvents` aufgerufen. Im Window zwischen Register und Replay koennte ein Live-Event zeitlich vor einem Replay-Event ankommen. Ergebnis: Out-of-Order-Events beim Reconnect-Client.

### Bewertung der WICHTIG-Finding

**Severity: AKZEPTABEL (Minor, out-of-scope)** — Begruendung per Gemini Runde 2:

- Kontext ist Monitoring-Stream, kein Trading-Entscheidungspfad (kein finanzielles Risiko)
- Aktuelle Implementierung garantiert At-Least-Once-Delivery (kein Event verloren)
- Gegenalternative (addEmitter NACH replayEvents) wuerde echten Datenverlust riskieren
- Lock-basierte Loesung wuerde Netzwerk-I/O in kritischem Pfad blockieren
- **Empfehlung Gemini:** Frontend-Side Deduplication via `eventId`-Tracking (best practice fuer Monitoring-Streams)
- Erfasst als offener Punkt #5 (Frontend-Aufgabe, ODIN-0xx)

**Gesamtbewertung: APPROVED** — Code ist produktionstauglich fuer Commit.

---

## 5. Offene Punkte (Out of Scope fuer ODIN-037)

| # | Thema | Prioritaet | Story-Kandidat |
|---|---|---|---|
| 1 | `instrumentId`-Validierung in StreamController (`@Pattern`-Validator oder Whitelist gegen konfigurierte Instrumente) | Niedrig | ODIN-0xx |
| 2 | `SseRingBuffer.add()` nutzt `Instant.now()` — Clock-Injektion wuerde testbarere Expiry-Tests ermoeglichen | Niedrig | ODIN-0xx |
| 3 | Echte SSE Event-Delivery-Verifikation (nicht nur Count-Asserts) — benoetigt echten HTTP-Dispatcher oder Spring Boot Test | Mittel | ODIN-0xx (E2E-Test) |
| 4 | NaN/Infinity-Guards in numerischen DTOs | Niedrig | ODIN-0xx |
| 5 | Frontend-Side SSE-Deduplication via `eventId`-Tracking (Gegenmassnahme fuer At-Least-Once Out-of-Order beim Reconnect) | Niedrig | ODIN-0xx (Frontend) |

---

## 6. Dateien

### Neue Dateien

- `odin-app/src/main/java/de/its/odin/app/dto/sse/PipelineStateEvent.java`
- `odin-app/src/main/java/de/its/odin/app/dto/sse/PositionUpdateEvent.java`
- `odin-app/src/main/java/de/its/odin/app/dto/sse/IndicatorUpdateEvent.java`
- `odin-app/src/main/java/de/its/odin/app/dto/sse/LlmAnalysisEvent.java`
- `odin-app/src/main/java/de/its/odin/app/dto/sse/TradeExecutedEvent.java`
- `odin-app/src/main/java/de/its/odin/app/dto/sse/AlertEvent.java`
- `odin-app/src/main/java/de/its/odin/app/dto/sse/DegradationChangeEvent.java`
- `odin-app/src/main/java/de/its/odin/app/dto/sse/CycleUpdateEvent.java`
- `odin-app/src/test/java/de/its/odin/app/dto/sse/SseEventDtoSerializationTest.java`
- `odin-app/src/test/java/de/its/odin/app/sse/SseEventTypeTest.java`
- `odin-app/src/test/java/de/its/odin/app/sse/SseStreamIntegrationTest.java`
- `odin-app/src/test/java/de/its/odin/app/sse/HeartbeatConfigTest.java`

### Geaenderte Dateien

- `odin-app/src/main/java/de/its/odin/app/sse/SseEventType.java` — Utility-Klasse → ENUM
- `odin-app/src/main/java/de/its/odin/app/sse/SseEmitterManager.java` — Memory-Leak-Fix, @PreDestroy, Exception-Handling
- `odin-app/src/main/java/de/its/odin/app/controller/StreamController.java` — pipelineStream ohne Timeframe
- `odin-app/src/test/java/de/its/odin/app/sse/SseEmitterManagerTest.java` — neue Tests fuer Stream-Key-Cleanup
- `odin-app/src/test/java/de/its/odin/app/sse/SseRingBufferTest.java` — neue Tests fuer Expiry und Invarianten
- `odin-app/src/test/java/de/its/odin/app/controller/StreamControllerTest.java` — auf pipelineStream umgeschrieben
- `odin-app/src/test/java/de/its/odin/app/controller/ExchangeControllerTest.java` — Pre-existing Fix: CoreProperties.SessionProperties
- `odin-app/src/test/java/de/its/odin/app/controller/TradingRunControllerTest.java` — Pre-existing Fix: CoreProperties.SessionProperties
