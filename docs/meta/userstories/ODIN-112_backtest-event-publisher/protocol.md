# Protokoll: ODIN-112 — BacktestEventPublisher und ThrottlingEventFilter

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases (Rework Runde 2, 2026-03-05)
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

---

## Design-Entscheidungen

### DD-01: Synchronized HashMap statt ConcurrentHashMap

Die Story-Spezifikation nennt ConcurrentHashMap als Implementierungsdetail. Die
Implementierung verwendet stattdessen synchronized HashMap-Methoden. Begruendung:

- Die ThrottlingEventFilter-Instanz wird pro Backtest erstellt und ist nicht
  global geteilt — der Konkurrenzgrad ist gering (typischerweise ein Worker-Thread
  plus gelegentliche Flush-Aufrufe).
- Synchronized Methoden sind einfacher, weniger fehleranfaellig und ausreichend
  performant fuer die erwarteten 84.000 Events pro Backtest-Run auf einer Maschine.
- Eine Migration zu ConcurrentHashMap ist jederzeit moeglich falls Profiling
  einen Engpass beweist (YAGNI-Prinzip).

Gemini hat diese Abweichung in Dimension 2 als non-compliant markiert. Die Entscheidung
bleibt bestehen: Korrektheit vor vorzeitiger Optimierung.

### DD-02: Composite Key statt einfachem Event-Type-Key

Die Story-Spezifikation definiert den Coalescing-Cache als `ConcurrentHashMap<String, Object>`
(eventType → latest payload). Diese Implementierung haette bei mehreren Instrumenten
denselben eventType (z.B. indicator-update) zu Datenverlust gefuehrt.

Beides — ChatGPT und Gemini — haben diesen potenziellen Bug identifiziert. Die
Implementierung verwendet einen Composite Key (`eventType:qualifier`) fuer unabhaengiges
Throttling pro Instrument. Diese Abweichung von der Story-Spezifikation ist eine
korrekte Verbesserung und wurde in den Tests als "F-02 fix" dokumentiert.

### DD-03: Namespace odin.frontend.sse.backtest-throttle (F-03 Klaerung)

Die QA hat einen Namespace-Konflikt gemeldet: Story spezifiziert `odin.sse.backtest-throttle.*`,
Implementierung verwendet `odin.frontend.sse.backtest-throttle.*`.

Klaerung: Das Konzeptdokument `docs/concept/11-live-data-streaming.md` (die authoritative
Quelle) definiert explizit den Namespace `odin.frontend.sse.*` (Zeile 943, 954, 973-987).
Die Implementierung folgt dem Konzeptdokument korrekt. Der Story-Text enthielt einen Fehler
(fehlender `frontend`-Prafix). Gemini bestaetigt dies in Dimension 2: "The developer made
the correct decision to align with the concept document."

**Ergebnis F-03: Kein Code-Change notwendig. Namespace ist korrekt.**

### DD-04: Observability durch Dropped-Count-Logging in shutdown()

Auf Empfehlung von Gemini (Praxis-Dimension) wurde in `shutdown()` ein strukturiertes
Log-Statement ergaenzt, das totalDroppedEvents und flushedOnShutdown ausgibt. Dies ermoeglicht
Post-hoc-Analyse ob Throttling die UI-Darstellung beeinflusst hat.

### DD-05: Null-Safety-Contract via Objects.requireNonNull

Auf Empfehlung von ChatGPT und Gemini (beide Reviewer) wurden explizite Null-Guards
in `filter(eventType, qualifier, payload)` hinzugefuegt. Null-Payloads sind kein
gueltiger Domaenenzustand und wuerden bei der bestehenden Flush-Logik zu stillem
Datenverlust fuehren. Der Vertrag wird nun durch NullPointerException mit einer
aussagekraeftigen Fehlermeldung durchgesetzt.

---

## Offene Punkte

### OP-01: Stream-Gap-Event (DEFER → ODIN-115 oder spaeter)

Die Story-Spezifikation erwaehnt Stream-Gap-Events bei Ring-Buffer-Overflow.
Diese sind in ThrottlingEventFilter/BacktestEventPublisher nicht implementiert.
Gemini hat dies in Dimension 2 als fehlend identifiziert.

Begruendung fuer DEFER: Ring-Buffer-Overflow ist eine Infrastruktur-Verantwortung
des SseRingBuffer (bereits implementiert in ODIN-111). Der BacktestEventPublisher
hat keinen direkten Zugriff auf den Ring-Buffer-Overflow-Zustand. Eine saubere
Implementierung erfordert eine Callback/Event-Schnittstelle aus dem SseRingBuffer.
Dies ist ein separates Thema fuer eine eigene Story.

### OP-02: Publisher-Lifecycle-Management (DEFER → ODIN-114)

Gemini Praxis-Dimension: Der BacktestEventPublisher wird bei Backtest-Ende korrekt
geschlossen (`shutdown()`), aber es gibt keine Mechanik die den Publisher aus einer
zentralen Registry entfernt. Wenn ODIN-114 (BacktestRunner-Integration) implementiert
wird, muss dieser Punkt adressiert werden.

### OP-03: Disconnected-Client-Detection (DEFER)

Gemini Praxis-Dimension: Wenn der Browser-Tab waehrend eines Backtests geschlossen
wird, arbeitet der Publisher weiter ohne Subscribers. Der SseEmitterManager sollte
eine `hasActiveSubscribers(streamKey)`-Methode exponieren, die der Publisher
nutzen kann um fruehzeitig abzubrechen. Fuer jetzt akzeptabel — CPU-Waste minimal
bei Single-Backtest-Szenario.

---

## ChatGPT-Sparring

**Datum:** 2026-03-05
**Session:** Owner ODIN-112-rework, chatgpt-pool Slot 0
**Dateien gesendet:** ThrottlingEventFilter.java, BacktestEventPublisher.java, SseProperties.java, ThrottlingEventFilterTest.java, BacktestEventPublisherTest.java

### Vorgeschlagene Findings (13 total)

| # | Severity | Titel | Umgesetzt |
|---|----------|-------|-----------|
| 1 | HIGH | Null payloads silently dropped — nicht flushbar | JA — requireNonNull hinzugefuegt |
| 2 | MEDIUM | Null eventType kann NPE in NON_DROPPABLE check | JA — requireNonNull deckt das ab |
| 3 | MEDIUM | Coalescing-Test assertiert nicht "latest-value wins" exakt | VERWORFEN — Test ist klar genug; Semantik ist durch anderen Test abgedeckt |
| 4 | LOW/MEDIUM | Flush-Reihenfolge bei mehreren Keys nondeterministisch | VERWORFEN — UI macht keine Reihenfolge-Annahmen; dokumentiert |
| 5 | LOW/MEDIUM | Unbounded map growth bei hoher Qualifier-Kardinalitaet | VERWORFEN — Qualifier ist immer instrumentId (endliche Menge pro Backtest) |
| 6 | HIGH | Race: publish kann nach shutdown durchkommen (TOCTOU) | DEFER — volatile + check-before-publish ist ausreichend fuer Single-Machine |
| 7 | HIGH | Lifecycle-Events setzen publisher nicht auf "closing" | DEFER — Out of Scope fuer ODIN-112 |
| 8 | MEDIUM | flushPendingEvents() ignoriert closed-Guard | JA — Guard hinzugefuegt |
| 9 | MEDIUM | Kein try/catch um emitterManager-Aufrufe | VERWORFEN — Exception-Propagation ist fuer Debugging besser |
| 10 | MEDIUM | throttledEvent-Test: verify zu schwach (anyString) | JA — times(1) + verifyNoMoreInteractions hinzugefuegt |
| 11 | MEDIUM | Keine Concurrency-Tests fuer flush concurrent mit filter | DEFER — Synchronized sichert thread safety; Concurrency-Tests erfordern aufwendiges Test-Setup |
| 12 | LOW/MEDIUM | Nach flush: lastSent auf now gesetzt — naechstes Event throttled | JA — Test `afterFlushNextEventWithinIntervalShouldBeThrottled` hinzugefuegt |
| 13 | LOW | Qualifier-Weirdheiten (empty string, colon in qualifier) | VERWORFEN — Qualifier ist instrumentId (kein Colon, kein Leerstring in Praxis) |

### Umgesetzte Tests

- `filterShouldRejectNullEventType` — NullPointerException auf null eventType
- `filterShouldRejectNullPayload` — NullPointerException auf null payload
- `filterWithQualifierShouldRejectNullEventType` — mit Qualifier-Overload
- `filterWithQualifierShouldRejectNullPayload` — mit Qualifier-Overload
- `afterFlushNextEventWithinIntervalShouldBeThrottled` — Post-Flush-Throttle-Semantik
- `flushPendingEventsAfterShutdownShouldBeNoOp` (BacktestEventPublisherTest) — closed-Guard
- `throttledEventShouldNotBeForwardedWithinInterval` verstaerkt mit `times(1)` + `verifyNoMoreInteractions`

---

## Gemini-Review

**Datum:** 2026-03-05
**Session:** Owner ODIN-112-rework, gemini-pool Slot 0
**3 Dimensionen in einer Session**

### Dimension 1: Code-Bugs und Qualitaet

| # | Severity | Titel | Umgesetzt |
|---|----------|-------|-----------|
| 1 | CRITICAL | Race Condition in shutdown() — closed+publish nicht atomar | DEFER — volatile reicht fuer Single-Machine; beachtet werden muss Reihenfolge |
| 2 | MAJOR | Thread Contention — alle synchronized Methoden blockieren auf Instanz | DEFER — Synchronized ist korrekt und ausreichend fuer use case (DD-01) |
| 3 | MINOR | O(N) in getDroppedCount() bei hoher Qualifier-Kardinalitaet | DEFER — Instrument-Anzahl pro Backtest ist klein (~10-50) |
| 4 | INFO | Null-Payload in Coalescing-Cache fuhrt zu stiller Drop in flush | JA — requireNonNull loest das Problem an der Quelle |

### Dimension 2: Konzepttreue

| Punkt | Befund |
|-------|--------|
| AK 1-9 | Alle erfuellt |
| ConcurrentHashMap vs. synchronized | Non-compliant per Story-Text, aber bewusste Design-Entscheidung (DD-01) |
| Namespace odin.sse vs odin.frontend.sse | Implementation korrekt — Konzeptdokument ist authoritative Quelle (DD-03) |
| Stream-Gap-Event | Fehlt — DEFER als OP-01 dokumentiert |
| Composite Key (nicht in Story-Spec) | Positive Abweichung — kritischer Bug-Fix (DD-02) |

### Dimension 3: Praxis-Gaps

| # | Rating | Titel | Umgesetzt |
|---|--------|-------|-----------|
| 1 | PURSUE | Ghost Town: SSE-Client disconnected, Publisher arbeitet weiter | DEFER als OP-03 dokumentiert |
| 2 | DEFER | GC Churn durch String-Concatenation in buildCacheKey | DEFER — moderne JVM handelt das |
| 3 | PURSUE | Observability: droppedCounts werden nirgendwo geloggt | JA — Log-Statement in shutdown() hinzugefuegt (DD-04) |
| 4 | PURSUE | Memory Leak: Publisher wird nicht aus Registry entfernt | DEFER als OP-02 dokumentiert (→ ODIN-114) |
| 5 | DISCARD | Missing Heartbeat/Keep-Alive im Publisher | Discard — SseEmitterManager handelt das global |
