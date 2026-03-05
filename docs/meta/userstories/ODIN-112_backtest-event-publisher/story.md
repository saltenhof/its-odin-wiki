## Kontext

Der BacktestEventPublisher ist der zentrale Adapter, der Backtest-Events an den SSE-Stream routet. Der ThrottlingEventFilter verhindert Event-Flooding bei schnellen Backtests (bis zu 84.000 Events pro Lauf). Zusammen bilden sie die Backpressure-Schicht, die sicherstellt, dass der Browser nicht ueberflutet und der Backtest-Runner nicht durch SSE-Sends blockiert wird.

## Modul

odin-app

## Abhaengigkeiten

#111 — ODIN-111: Backtest-SSE DTOs und EventType-Erweiterung (DTOs muessen existieren)

## Scope

### In Scope
- `BacktestEventPublisher`: Implementiert `MonitoringEventPublisher`, remappt instrumentId → `backtest-{id}` Stream-Key, delegiert an SseEmitterManager
- `ThrottlingEventFilter`: Wall-Clock-basiertes Throttling mit konfigurierbaren Intervallen pro Event-Typ, optional SimTime-basiert
- Three-Tier Drop-Policy: NON-DROPPABLE / THROTTLE / DROPPABLE
- Coalescing: Latest-Value-Cache fuer Snapshot-Events (indicator-update, position-update)
- Stream-Gap-Event Emission bei Ring-Buffer-Overflow
- Flush-on-Pause: Pending throttled values flushen bei Day-Completion

### Out of Scope
- Async Dispatch Layer (Story B3 / ODIN-113)
- Pipeline-Wiring / BacktestRunner-Integration (Story B4 / ODIN-114)
- Frontend-Aenderungen

## Akzeptanzkriterien
- [ ] BacktestEventPublisher implementiert MonitoringEventPublisher Port
- [ ] Stream-Key Remapping: instrumentId → "backtest-{backtestId}" funktioniert korrekt
- [ ] ThrottlingEventFilter droppt Events korrekt nach konfiguriertem Intervall
- [ ] NON-DROPPABLE Events (trade-executed, pipeline-state, day-summary) werden nie gedroppt
- [ ] Coalescing: Bei mehreren indicator-updates innerhalb eines Intervalls wird nur der letzte gesendet
- [ ] Flush-on-Pause: Bei Day-Completion werden alle gecachten Werte sofort gesendet
- [ ] Unit-Tests fuer alle Throttling-Szenarien (Timing, Edge Cases)
- [ ] Unit-Tests fuer Coalescing (latest wins)
- [ ] Build kompiliert fehlerfrei

## Technische Details

**Neue Klassen:**

1. **BacktestEventPublisher** in `de.its.odin.app.sse`
   - Implementiert `de.its.odin.api.port.MonitoringEventPublisher`
   - Konstruktor: `BacktestEventPublisher(String backtestId, SseEmitterManager emitterManager, ThrottlingEventFilter filter)`
   - Stream-Key: `"backtest-" + backtestId` (nicht instrumentId)
   - Alle publish-Methoden delegieren an `filter.accept(event)` → bei Durchlass an `emitterManager.sendEvent(streamKey, event)`
   - Lifecycle-Events (completed, failed, cancelled) bypassen den Filter

2. **ThrottlingEventFilter** in `de.its.odin.app.sse`
   - Konfigurierbare Intervalle pro Event-Typ (aus SseProperties)
   - Three-Tier Drop-Policy:
     - **NON_DROPPABLE:** trade-executed, pipeline-state, backtest-day-summary, alert, completed, failed, cancelled — werden IMMER durchgelassen
     - **THROTTLE:** indicator-update, position-update, backtest-bar-progress, cycle-update — Intervall-basiert, mit Coalescing
     - **DROPPABLE:** backtest-quant-score, backtest-gate-result — aggressiveres Throttling
   - Coalescing-Cache: `ConcurrentHashMap<String, Object>` (eventType → latest payload) — bei Flush wird der letzte Wert gesendet
   - `lastSent`-Timestamps: `ConcurrentHashMap<String, Long>` (eventType → System.nanoTime()) — thread-safe
   - `flush()` Methode: Sendet alle gecachten Werte sofort (aufgerufen bei Day-Completion)
   - Stream-Gap-Event: Wenn Ring-Buffer-Overflow erkannt wird, ein synthetisches `stream-gap` Event emittieren

**Konfiguration (SseProperties erweitern):**
- `odin.sse.backtest-throttle.indicator-update=200ms`
- `odin.sse.backtest-throttle.position-update=500ms`
- `odin.sse.backtest-throttle.bar-progress=1000ms`
- `odin.sse.backtest-throttle.quant-score=2000ms`
- `odin.sse.backtest-throttle.gate-result=2000ms`

**Referenz:** Concept 11 §6.4 (Drop-Policy Tabelle), §6.5 (Producer-Entkopplung), §6.6 (Wall-Clock vs SimClock), §9.4 (Flush-on-Pause), §9.5 (SimTime Throttling)

## Konzept-Referenzen
- `docs/concept/11-live-data-streaming.md` — §6.4 Drop-Policy (Three-Tier-Tabelle mit allen Event-Zuordnungen)
- `docs/concept/11-live-data-streaming.md` — §6.5 Producer-Entkopplung (Bounded Queue, Overflow-Handling)
- `docs/concept/11-live-data-streaming.md` — §6.6 Wall-Clock vs. SimClock Throttling
- `docs/concept/11-live-data-streaming.md` — §8 Dual-Use-Architektur (MonitoringEventPublisher als Adapter)
- `docs/concept/11-live-data-streaming.md` — §9.4 Flush-on-Pause (Day-Completion Flush-Semantik)
- `docs/concept/11-live-data-streaming.md` — §9.5 SimTime Throttling (optional, Wall-Clock als Default)

## Guardrail-Referenzen
- `CLAUDE.md` — Port-Abstraktion: gegen `de.its.odin.api.port.MonitoringEventPublisher` programmieren
- `CLAUDE.md` — MarketClock verwenden, nicht `Instant.now()` in Trading-Code-Pfaden
- `CLAUDE.md` — @ConfigurationProperties als Record + @Validated
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

## Notizen fuer den Implementierer

- Die ThrottlingEventFilter-Logik **muss thread-safe** sein — ConcurrentHashMap fuer lastSent-Timestamps und Coalescing-Cache. Backtests laufen potenziell auf Worker-Threads.
- Der Coalescing-Cache ist bewusst simpel: `Map<String, Object>` (eventType → latest payload). Kein Ring-Buffer, kein History — nur der letzte Wert pro Typ.
- **Flush-on-Pause** ist kritisch: Nach jedem simulierten Handelstag muessen alle gecachten Werte rausgehen, bevor der naechste Tag startet. Sonst gehen End-of-Day-Werte verloren.
- Wall-Clock-Throttling ist der Default. SimTime-Throttling (basierend auf simulierter Marktzeit statt Echtzeit) ist optional und kann spaeter ergaenzt werden — aber die Schnittstelle sollte es bereits vorsehen.
- Stream-Gap-Events sind ein Signal an die UI, dass Events uebersprungen wurden. Format: `{"type":"stream-gap","droppedCount":42,"oldestDropped":"2025-01-15T14:30:00Z"}`.

## Definition of Done
- [ ] Code kompiliert fehlerfrei
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Surefire: `*Test`)
- [ ] Integrationstests (Failsafe: `*IntegrationTest`)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
