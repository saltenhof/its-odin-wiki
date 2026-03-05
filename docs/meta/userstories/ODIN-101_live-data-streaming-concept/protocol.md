# ODIN-101 — Implementierungsprotokoll

## Konzept-Erstellung

### Vorgehen
- Codebase-Analyse (3 parallele Sub-Agents): BacktestRunner, TradingPipeline, Controllers, Event-System, API-Ports, Konzeptdocs
- Konzept-Dokument geschrieben: `docs/concept/11-live-data-streaming.md` (944 Zeilen vor Review-Einarbeitung)

### Bestandsaufnahme (Key Finding)
Die SSE-Infrastruktur war bereits deutlich weiter als in der Story angenommen:
- SseEmitterManager mit MonitoringEventPublisher-Port
- StreamController mit 3 Endpoints (pipeline, global, backtest)
- SseRingBuffer mit Last-Event-ID Replay
- 13 SseEventTypes, 8 DTOs
- **Luecke**: Backtest-Stream sendet nur progress/completed/failed/cancelled — keine Live-Trading-Events

## Multi-LLM-Review

### ChatGPT-Review
**Fokus:** Architektur-Robustheit, Failure Modes, Backpressure

Kritische Findings (eingearbeitet):
1. **Producer-Decoupling:** `sendEvent()` darf Producer-Thread nicht blockieren → Async Dispatch Layer mit Bounded Queue pro Stream + Single Dispatcher Thread (Abschnitt 6.5)
2. **NON-DROPPABLE nicht end-to-end:** Ring-Buffer-Overflow kann NON-DROPPABLE Events ueberschreiben → EventLog als durables Fallback, REST-Endpoint als Late-Join-Mechanismus (Abschnitt 6.4)
3. **Coalescing statt Dropping:** Fuer Snapshot-Events (indicator-update, position-update) Latest-Value-Cache mit kontrollierter Kadenz statt reinem Time-based-Dropping (Abschnitt 6.4)
4. **Replay-Reihenfolge:** Emitter wird vor Replay attached → Out-of-Order-Delivery → Replay-First als Implementierungsanforderung (Abschnitt 7.6)
5. **Stream-Gap-Signalisierung:** `stream-gap` Control-Event bei Ring-Buffer-Overflow oder TTL-Expiry, damit Client Luecken erkennt (Abschnitt 7.5)
6. **Wall-Clock vs SimClock fuer TTL:** Ring-Buffer-Retention nutzt Wall-Clock, Event-Payloads enthalten SimClock-marketTime (Abschnitt 6.6)

### Gemini-Review
**Fokus:** UX-Konsistenz, Frontend-Impact, Timing-Semantik

Kritische Findings (eingearbeitet):
1. **NON-DROPPABLE Widerspruch** (Ueberlappung mit ChatGPT Finding #2) — gemeinsam adressiert
2. **Stale UI nach Throttle:** Flush-on-Pause Mechanismus — bei Tagesabschluss, Pause und Backtest-Ende werden alle gethrottleten Werte sofort geflusht (Abschnitt 9.4)
3. **Frontend Memory Leaks:** Bei langen Streams droht Speicherueberlauf im Browser → Memory Windowing Anforderung, Verweis auf ODIN-102 (Abschnitt 10.5)
4. **SimTime-basiertes Throttling:** Alternative zum Wall-Clock-Throttling — Filterung nach simulierten Bar-Counts fuer konsistente visuelle Aufloesung, als Konfigurationsoption (Abschnitt 9.5)
5. **Multi-Instrument Interleaving:** Bereits in Q4 der offenen Fragen adressiert (chronologisch nach SimClock-Time, Frontend filtert nach instrumentId)

### Bewertung
Beide Reviews bestaetigen die architektonische Grundentscheidung (SSE, Dual-Use, Ring-Buffer).
Die Kritikpunkte betreffen Robustheit und Edge Cases, nicht das Grunddesign.
Alle 9 substanziellen Findings wurden ins Konzept eingearbeitet.

## Ergebnis
- Konzept-Dokument: `docs/concept/11-live-data-streaming.md`
- Umfang: 1026 Zeilen (nach Review-Einarbeitung, vorher 944)
- Eingearbeitete Aenderungen: 6 neue Abschnitte (6.5, 6.6, 7.5, 9.4, 9.5, 10.5), 2 erweiterte Abschnitte (6.4, 7.6)
- Status: Abgeschlossen
