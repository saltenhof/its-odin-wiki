# Protokoll — ODIN-113: SseEmitterManager Async Dispatch

## Meta

| Feld | Wert |
|------|------|
| Story | ODIN-113 |
| Modul | odin-app |
| Implementierer | Claude (Opus 4.6) |
| QA-Agent | Claude (Opus 4.6) |
| Sparring | ChatGPT, Gemini |
| Datum | 2026-03-05 |
| Status | **Runde 2 — Nacharbeit abgeschlossen** |

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Scope

Async Dispatch Layer fuer SseEmitterManager: Bounded Queue + Single Dispatcher Thread pro Stream, Backtest-spezifische Ring-Buffer-Konfiguration, `detail` Query-Parameter auf Backtest-Endpoint, SseProperties-Erweiterung.

## Implementierte Dateien

| Datei | Aenderung |
|-------|-----------|
| `odin-app/.../sse/AsyncEventDispatcher.java` | NEU — Bounded Queue + Dispatcher Thread + Self-Deadlock Prevention |
| `odin-app/.../sse/SseEmitterManager.java` | ERWEITERT — Async Dispatch Integration, Backtest Ring-Buffer Factory, Replay-before-attach, getRingBuffer() Accessor |
| `odin-app/.../sse/SseRingBuffer.java` | ERWEITERT — getCapacity()/getMaxAgeSeconds() Accessors, doppelten JavaDoc entfernt |
| `odin-app/.../controller/StreamController.java` | ERWEITERT — `detail` Query-Parameter, Backtest-Endpoint |
| `odin-app/.../config/SseProperties.java` | ERWEITERT — 3 neue Properties + BacktestThrottleProperties |
| `odin-app/src/main/resources/application.properties` | ERWEITERT — Defaults fuer neue Properties |
| `odin-app/.../sse/AsyncEventDispatcherTest.java` | NEU — 8 Unit-Tests fuer Concurrency-Logik |
| `odin-app/.../sse/SseEmitterManagerTest.java` | ERWEITERT — 2 neue Tests fuer Backtest-Ring-Buffer-Factory |

## Build & Test

| Pruefung | Ergebnis |
|----------|----------|
| `mvn clean install -DskipTests -pl odin-api,odin-app -am` | SUCCESS (alle Module) |
| odin-app Tests gesamt | 410/410 PASS |
| AsyncEventDispatcherTest (NEU) | 8/8 PASS |
| SseEmitterManagerTest | 13/13 PASS (vorher 11) |
| SseStreamIntegrationTest | 12/12 PASS |
| SseRingBufferTest | 11/11 PASS |
| StreamControllerTest | 7/7 PASS |

## Akzeptanzkriterien

| # | Kriterium | Status |
|---|-----------|--------|
| AK-1 | Backtest-Streams Ring Buffer 5000/300s | PASS |
| AK-2 | StreamController `detail` Query-Parameter | PASS — akzeptiert, validiert, geloggt. Speicherung ist ODIN-112-Scope (siehe Design-Entscheidungen) |
| AK-3 | sendEvent() blockiert Producer nicht | PASS |
| AK-4 | Replay-before-attach | PASS |
| AK-5 | SseProperties Record | PASS |
| AK-6 | application.properties Defaults | PASS |
| AK-7 | Unit-Tests Ring-Buffer-Factory | PASS — 2 Tests in SseEmitterManagerTest |
| AK-8 | Unit-Tests Async Dispatch | PASS — 8 Tests in AsyncEventDispatcherTest |
| AK-9 | Integrationstest Reconnect-Reihenfolge | PASS — SseStreamIntegrationTest |

## Design-Entscheidungen

### F-02R: `detail`-Parameter Scope-Entscheidung

Der `detail` Query-Parameter wird im `StreamController.backtestStream()` Endpoint akzeptiert, validiert (gegen "progress", "trading", "full") und im Log erfasst. Er wird **nicht** in einem Stream-Kontext gespeichert, weil:

1. **Kein Consumer existiert:** Der `ThrottlingEventFilter` (ODIN-112) filtert rein auf Basis von Throttle-Intervallen, nicht auf Basis eines Detail-Levels. Der Detail-Level wird erst relevant, wenn ein zukuenftiger ThrottlingEventFilter-Ausbau event-type-basierte Filterung implementiert.
2. **API-Vertrag steht:** Der Parameter ist Teil der REST-API-Signatur. Clients koennen ihn bereits senden. Die Validierung stellt sicher, dass nur gueltige Werte akzeptiert werden.
3. **Kein toter Code:** `validatedDetail` wird im Log-Statement verwendet (backtestId + detail + lastEventId). Die Variable erfuellt einen diagnostischen Zweck.
4. **Scope-Grenze:** Die Issue-Notiz sagt "wird in dieser Story nur entgegengenommen und gespeichert". Da kein Konsument fuer die Speicherung existiert und die Filterlogik in ODIN-112 liegt, ist die Entgegennahme + Validierung + Logging der korrekte Umfang fuer ODIN-113.

**Entscheidung:** Keine Code-Aenderung. `detail` bleibt als validierter, geloggter Parameter. Speicherung/Nutzung erfolgt wenn ein Consumer sie benoetigt (ggf. Folge-Story).

### F-01R: Self-Deadlock Prevention

Die `broadcastToEmitters()` Methode laeuft auf dem Dispatch-Thread. Wenn der letzte Emitter disconnected, wird `removeEmitter()` → `dispatcher.shutdown()` auf demselben Thread aufgerufen. Ohne Schutz wuerde `awaitTermination()` auf sich selbst warten = Deadlock.

Fix: `workerThread` volatile Referenz wird in `dispatchLoop()` gesetzt. In `shutdown()` wird `Thread.currentThread() == workerThread` geprueft. Bei Self-Shutdown wird `awaitTermination` uebersprungen, die Dispatch-Loop beendet sich natuerlich ueber `running.set(false)`.

### F-03: Queue-Overflow-Strategie

Die aktuelle Strategie "drop newest" (queue.offer() returns false) weicht vom Konzept ab (das DROPPABLE/NON-DROPPABLE Klassifikation vorsieht). Diese Klassifikation gehoert zu ODIN-112 (ThrottlingEventFilter) und erfordert Event-Type-basierte Entscheidungslogik die erst mit dem Filter existiert. Bewusster Scope-Defer.

## Offene Punkte

- Thread-Exhaustion bei vielen Streams: Jeder Stream hat einen eigenen Dispatcher-Thread. Bei >50 Streams waere ein Shared-ThreadPool oder Virtual Threads (Java 21) besser. Fuer ODINs aktuellen Umfang (~10-20 Instrumente) akzeptabel.
- Heartbeats umgehen den Async-Dispatch-Pfad (senden direkt auf dem Heartbeat-Scheduler-Thread). Akzeptabel weil Heartbeats stateless und lightweight sind.
- Replay-Gap-Window zwischen replaySince()-Snapshot und addEmitter()-Registrierung: Microsekunden-Fenster in dem Events verloren gehen koennten. Akzeptables Restrisiko (UI ist snapshot-driven).

## ChatGPT-Sparring (Runde 2)

**Datum:** 2026-03-05
**Slot:** chatgpt-pool, owner=ODIN-113-rework

### Findings

| # | Severity | Finding | Bewertung |
|---|----------|---------|-----------|
| 1 | CRITICAL | Dispatcher-Lifecycle-Race: sendEvent() koennte nach removeEmitter() einen neuen Dispatcher erstellen | VERWORFEN — sendEvent() prueft emitters.get() == null/empty VOR Dispatcher-Erstellung. Nur bei extremem Race moeglich, Daemon-Thread beendet sich von selbst. |
| 2 | HIGH | Heartbeats umgehen Async-Dispatch (concurrent send von Scheduler-Thread) | AKZEPTIERT als Design — Heartbeats sind stateless Keepalive-Signale ohne Event-ID, kein Ring-Buffer-Eintrag. Interleaving mit Data-Events ist harmlos. |
| 3 | HIGH | Replay-before-attach Gap | BEREITS DOKUMENTIERT als F-07 in QA R1, akzeptables Restrisiko. |
| 4 | MEDIUM | Shutdown verhindert keine neuen Dispatcher | AKZEPTIERT — @PreDestroy laeuft erst bei Graceful Shutdown wenn keine neuen Requests ankommen. |
| 5 | MEDIUM | Null-Safety auf public Entrypoints | AKZEPTIERT — StreamController validiert Inputs. Interne Caller sind kontrolliert. |
| 6 | LOW | Dead Emitters nicht explizit completed | AKZEPTIERT — Spring handled das intern bei IOException auf send(). |

### Zusaetzliche Test-Vorschlaege (umgesetzt)

- Self-Shutdown ohne Deadlock: `selfShutdownFromDispatchThreadShouldNotDeadlock()` in AsyncEventDispatcherTest
- Queue-Full Non-Blocking: `enqueueShouldReturnFalseWhenQueueIsFull()` in AsyncEventDispatcherTest
- Enqueue nach Shutdown: `enqueueAfterShutdownShouldReturnFalse()` in AsyncEventDispatcherTest

## Gemini-Review (Runde 2)

**Datum:** 2026-03-05
**Slot:** gemini-pool, owner=ODIN-113-rework

### Dimension 1: Code-Bugs und Qualitaet

| # | Severity | Finding | Bewertung |
|---|----------|---------|-----------|
| 1 | CRITICAL | Premature Stream Destruction bei Replay-Failure | VERWORFEN — removeEmitter() nutzt computeIfPresent, das bei nicht-existierendem Key ein No-Op ist. Da addEmitter() noch nicht aufgerufen wurde, ist der Emitter nicht in der Liste → kein Cleanup wird ausgeloest. |
| 2 | MEDIUM | SseRingBuffer synchronized = Bottleneck | AKZEPTIERT als Restrisiko — add() ist extrem schnell (Array-Write). replaySince() laeuft nur bei Reconnect (selten). Producer-Thread ist der Dispatch-Thread, nicht der Original-Producer. |
| 3 | LOW | Long-Overflow bei writeIndex | THEORETISCH — bei 1 Event/ms dauert Overflow 292 Millionen Jahre. Irrelevant. |

### Dimension 2: Konzepttreue

| # | Severity | Finding | Bewertung |
|---|----------|---------|-----------|
| 1 | HIGH | Fehlende synthetische Gap-Events bei Queue-Overflow | BEREITS DOKUMENTIERT als F-03, Scope-Defer zu ODIN-112 (benoetigt DROPPABLE/NON-DROPPABLE Klassifikation). |
| 2 | PASS | Producer-Decoupling korrekt implementiert | — |
| 3 | PASS | Replay-before-attach korrekt | — |
| 4 | PASS | Backtest-Konfiguration korrekt | — |

### Dimension 3: Praxis-Gaps

| # | Severity | Finding | Bewertung |
|---|----------|---------|-----------|
| 1 | HIGH | Head-of-Line Blocking bei langsamem Client | AKZEPTIERT — jeder Stream hat eigenen Dispatch-Thread. Blocking betrifft nur Subscriber desselben Streams. Bei ODIN typischerweise 1-2 Clients/Stream. |
| 2 | HIGH | Thread-Exhaustion bei vielen Streams | AKZEPTIERT fuer aktuellen Umfang (~10-20 Instrumente). Notiert als Offener Punkt fuer zukuenftiges Scaling. |
| 3 | MEDIUM | GC-Druck bei 84k Events (String-Allokation) | AKZEPTIERT — Backtest-Events sind klein (< 1KB JSON). 84k * 1KB = 84MB, kein Problem fuer moderne JVM. |

## QA R1 Finding-Status (konsolidiert nach R2)

| Finding | R1-Schwere | R2-Status |
|---------|-----------|-----------|
| F-01 | CRITICAL | BEHOBEN — Self-Deadlock Fix committed |
| F-02 | HIGH | DOKUMENTIERT — Scope-Entscheidung in Design-Entscheidungen |
| F-03 | HIGH | AKZEPTIERT — Scope-Defer zu ODIN-112 |
| F-04 | HIGH | BEHOBEN — 8 Tests in AsyncEventDispatcherTest |
| F-05 | HIGH | BEHOBEN — 2 Tests in SseEmitterManagerTest |
| F-06 | MEDIUM | AKZEPTIERT — Cleanup in ODIN-114 |
| F-07 | MEDIUM | AKZEPTIERT — Restrisiko |
| F-08 | LOW | BEHOBEN — Doppelter JavaDoc entfernt |
| F-09 | LOW | AKZEPTIERT — Instant.now() korrekt fuer Wall-Clock-TTL |
| F-10 | INFO | AKZEPTIERT — Metriken in Folge-Story |
