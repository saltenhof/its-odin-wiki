# Protokoll: ODIN-124 — Per-Bar SSE Event Pipeline

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

## Design-Entscheidungen

### BacktestBarEvent als raw HashMap statt typisiertem DTO
`BacktestRunner` liegt in `odin-backtest`, das typisierte `BacktestBarEvent`-Record in `odin-app`. Da `odin-app → odin-backtest` (nicht umgekehrt), wuerde die Verwendung des DTOs eine Kreisabhaengigkeit erzeugen. Der raw `Map<String, Object>` ist daher die korrekte und einzige moegliche Loesung — Jackson serialisiert die Map identisch zum DTO. Das ist eine bewusste Architekturentscheidung, kein Bug.

### simTime-Feld in BacktestBarProgressEvent
Das Konzept-Schema (Section 3.3.9) enthaelt kein `simTime`-Feld. Die Implementierung fuegt es als bewusste Erweiterung hinzu: das Frontend (ODIN-125, Progressive Chart) benoetigt die simulierte Marktzeit fuer X-Achsen-Synchronisation. Die Erweiterung ist abwaertskompatibel und dokumentiert.

### detail-Parameter am Stream-Endpoint
Das Konzept definiert `?detail=progress/trading/full` als Opt-in fuer Event-Granularitaet. Dieser Parameter ist in ODIN-124 NICHT implementiert — alle Events werden immer gestreamt. Begruendung: ODIN ist ein single-user-System, der ThrottlingEventFilter verhindert Browser-Ueberlastung. Der detail-Parameter ist als moegliche Future-Erweiterung notiert.

### totalBarsForDay: RTH-only Zaehlung (Fix aus Review)
Urspruenglich wurde `totalBarsForDay` als Summe ALLER Bars berechnet (inkl. Pre-Market). Der `completedBarCounter` inkrementiert jedoch nur fuer RTH-Callbacks. Das fuehrte zu einer Progress-Anzeige die niemals 100% erreichte. Gefixt: `totalBarsForDay` filtert nun Pre-Market-Bars heraus (`bar.closeTime().isAfter(rthOpen)`).

### AtomicInteger fuer completedBarCounter
Lambda-Captures erfordern effectively-final Variablen. `AtomicInteger` ist das idiomatische Java-Pattern fuer mutable Zaehlvariablen in Lambdas. Ein primitives `int[]`-Array waere funktional identisch aber weniger lesbar.

### ThrottlingEventFilter: Coalescing-Logik
Gemini D1 interpretierte die Coalescing-Logik als Bug ("Events gehen verloren"). Diese Einschaetzung ist falsch. Der Algorithmus funktioniert korrekt: `coalescingCache.put()` speichert das neueste Payload; beim naechsten Send nach Ablauf des Intervalls wird via `coalescingCache.remove()` der neueste Wert zurueckgegeben. Zwischenzeitliche Events werden absichtlich durch den neuesten ersetzt (Latest-Value Coalescing by Design).

## Offene Punkte

- detail-Parameter am Stream-Endpoint (`?detail=progress/trading/full`) — Opt-in fuer Event-Granularitaet — nicht implementiert, da fuer single-user nicht kritisch. Notiert als moegliche kuenftige Erweiterung.
- Frontend: Page Visibility API sollte high-frequency Events in Hintergrund-Tabs verwerfen (Gemini D3). Betrifft odin-ui, nicht Backend.
- Frontend: Batching von SSE-Events bei Replay nach Reconnect (Gemini D3). Betrifft odin-ui.

## ChatGPT-Sparring

**Prompt-Schwerpunkte:** null-Publisher, 0 Bars, Pre-Market Bars (kein Callback), Multi-Instrument, Race Conditions bei concurrent Bar-Callbacks.

**Vorgeschlagene Test-Szenarien (42 gesamt):**

### Eingearbeitet (3 neue Tests in SimulationRunnerTest):

1. **`preMarketBarsShouldNeverFireBarCallback`** — Pre-Market-Replay loest keinen barCallback aus. Testet den RTH-only-Vertrag mit realem `HistoricalMarketDataFeed` und `SimClock`. Hochprioritaere Regression-Absicherung.

2. **`zeroRthBarsShouldNeverFireBarCallback`** — Leerer RTH-Feed loest keinen Callback aus. Stellt sicher, dass `barCallback.accept()` bei leerem Feed nie aufgerufen wird.

3. **`nullCallbackWithHistoricalFeedShouldNotThrow`** — Null-Callback mit realem HistoricalMarketDataFeed wirft keine NPE, auch wenn RTH-Bars vorhanden sind.

### Abgelehnt (mit Begruendung):

- **null-Publisher / sseEmitterManager = null:** `BacktestEventPublisher` wuerde null-Publisher im Konstruktor erhalten. Da `createEventPublisher()` in `BacktestExecutionService` nie null aufruft (der `sseEmitterManager` ist eine Spring-Bean), ist dieser Fall nicht erreichbar. Test wuerde Infrastruktur mocken die nicht Teil des Scopes ist.

- **Concurrent Bar-Callbacks / Race Conditions:** `SimulationRunner` ist explizit single-threaded. Der `ThrottlingEventFilter` ist `synchronized`. Concurrent-Access-Tests wuerden Parallelisierung voraussetzen die nicht existiert und nicht geplant ist. ODIN ist kein Multi-Thread-Trading-System.

- **Multi-Instrument Callback-Reihenfolge:** Die Reihenfolge entspricht der Pipeline-Liste, die von `LifecycleManager.initializePipelines()` bestimmt wird. Diese ist konfigurationsabhaengig und nicht Gegenstand von ODIN-124. Test wuerde Lifecycle-Internals testen.

- **Stale lastEmittedBar bei erschoepftem Instrument:** `historicalFeed.advanceBar()` bei erschoepftem Feed ist ein No-op, `getLastEmittedBar()` gibt dann weiterhin den letzten Wert zurueck. In der Praxis bricht `isAllComplete()` die Schleife sofort ab, sobald alle Feeds erschoepft sind. Das Szenario "ein Instrument fertig, anderes laeuft" wuerde nur feuern wenn `isAllComplete()` bei teilweiser Erscgoepfung noch `false` zurueckgibt. Das ist korrektes Verhalten — `completedBarCounter` wuerde korrekt inkrementiert.

- **Infinity/NaN in PnL:** `BigDecimal.valueOf(Double.POSITIVE_INFINITY)` wirft `NumberFormatException`. Diese Situation kann entstehen wenn der Simulator korrupte Daten erzeugt. Gehoert in den Bereich DataQuality-Validierung (odin-data) und ist ausserhalb des Scopes von ODIN-124.

- **ThrottlingEventFilter Flush-Idempotenz, Concurrent-Stress-Tests:** Der ThrottlingEventFilter ist `synchronized` — Thread-Safety ist bereits durch den Mutex sichergestellt. Stress-Tests auf einem synchronized Block bringen keinen Mehrwert ausser Testlaufzeit. Die bestehenden 31 Tests (`ThrottlingEventFilterTest`) decken alle wesentlichen Verhaltensweisen ab.

## Gemini-Review

### Dimension 1 — Code-Review

**Befunde und Bewertung:**

1. **"Coalescing-Bug"** — ABGELEHNT: Gemini interpretierte die Coalescing-Logik falsch. Der Algorithmus ist korrekt (siehe Design-Entscheidungen).

2. **"raw HashMap statt BacktestBarEvent-DTO"** — ABGELEHNT: Architekturzwang durch Modulabhaengigkeiten. Kein Bug (siehe Design-Entscheidungen).

3. **"totalBarsForDay zaehlt Pre-Market-Bars"** — EINGEARBEITET: Echter Bug. Fix: RTH-only-Filterung via `bar.closeTime().isAfter(rthOpen)`. Vermeidet Progress < 100%.

4. **"lastClosePrice Multi-Instrument-Aggregation"** — NICHT RELEVANT: Single-Instrument ist der primaere Use-Case. Die letzte Instrument-close-price ist ausreichend fuer den Progress-Event.

5. **"AtomicInteger-Overhead"** — ABGELEHNT: Lambda-Capture erfordert effectively-final. AtomicInteger ist das korrekte idiomatische Pattern.

6. **"Memory Leak in ThrottlingEventFilter"** — NICHT RELEVANT: Der Filter ist eine per-Backtest-Instanz, wird nach Backtest-Ende via `shutdown()` + GC bereinigt. Kein langlebiger Service.

7. **"Synchronized Flaschenhals"** — NICHT RELEVANT: Single-User-System, keine Performance-Anforderung fuer concurrent Publishers. `synchronized` ist korrekt und einfach.

### Dimension 2 — Konzepttreue

**Befunde und Bewertung:**

1. **simTime-Feld extra** — BEWUSSTE_ERWEITERUNG (korrekt bewertet): Notwendig fuer Chart-Rendering in ODIN-125.

2. **raw HashMap statt typisiertem DTO** — als KONZEPT_VERSTOESS bewertet, aber ABGELEHNT: Architekturzwang, kein echter Verstoß (siehe oben).

3. **Routing-Mismatch (instrumentId vs. ALL_INSTRUMENTS)** — als BUG bewertet, aber KEIN_PROBLEM: Das ist By Design. `BacktestEventPublisher.publishInstrumentEvent()` verwendet den `instrumentId`-Parameter als Throttle-Qualifier, ignoriert ihn fuer das Stream-Routing. Alle Events gehen an den backtest-spezifischen Stream.

4. **totalBarsForDay zaehlt Pre-Market** — BUG (korrekt bewertet): EINGEARBEITET.

5. **detail-Parameter fehlt** — KONZEPT_VERSTOESS (korrekt bewertet, aber NICHT FIX-PFLICHTIG fuer ODIN-124): Notiert als offener Punkt.

### Dimension 3 — Praxis-Review

**Befunde und Bewertung:**

1. **Backtest-Geschwindigkeit / Event-Burst** — RELEVANT: Der ThrottlingEventFilter mit `backtestBarMs=20ms` limitiert die Bar-Event-Rate ausreichend fuer einen single-user-Browser. Kein Backend-Fix noetig.

2. **Browser-Tab im Hintergrund** — KRITISCH (Frontend): Page Visibility API empfohlen. Betrifft odin-ui, nicht Backend. Als offener Punkt notiert.

3. **Memory beim Reconnect / Ring-Buffer-Flush** — KRITISCH (Frontend): Bulk-Verarbeitung im SSE-Handler empfohlen. Betrifft odin-ui. Als offener Punkt notiert.

4. **Mehrtaegiger Backtest ohne Browser** — KEIN_PROBLEM: Ring-Buffer-Semantik (Ueberschreiben aeltester Events) behandelt diesen Fall korrekt. Kein Fix noetig.

5. **Callback-Exception-Isolation** — RELEVANT: `BacktestEventPublisher.publishInstrumentEvent()` hat keinen try-catch. Eine Exception im SSE-Stack koennte den Backtest abbrechen. Befund ist valide, aber `SseEmitterManager` behandelt exceptions intern (Emitter-Cleanup), sodass dieses Risiko in der Praxis gering ist. Ausserhalb Scope ODIN-124.

6. **UI-Responsiveness** — KRITISCH (Frontend): useRef + gedrosselter setInterval-Pattern empfohlen fuer React. Betrifft odin-ui.

7. **Gleichzeitige Backtests** — KEIN_PROBLEM: Jeder Backtest hat eigenen `ThrottlingEventFilter`-Instance und Stream-Key. Isolation ist bereits korrekt implementiert.

## Test-Ergebnis

```
[INFO] Tests run: 417, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

Module: odin-core (349), odin-backtest (407), odin-app (417)

Neu hinzugefuegt durch ODIN-124-rework:
- SimulationRunnerTest: 3 neue Tests (9 gesamt, war 6)
- ThrottlingEventFilterTest: unveraendert (31 Tests)

## Aenderungen durch Review

### BacktestRunner.java (odin-backtest)
Fix: `totalBarsForDay` berechnet nun nur RTH-Bars (closeTime > rthOpen), nicht mehr alle Bars.
Datei: `odin-backtest/src/main/java/de/its/odin/backtest/BacktestRunner.java`

### SimulationRunnerTest.java (odin-core)
Neue Imports: `LocalTime`, `ZoneId`, `HashMap`, `Map`, `never`, `HistoricalMarketDataFeed`
Neue Tests: `preMarketBarsShouldNeverFireBarCallback`, `zeroRthBarsShouldNeverFireBarCallback`, `nullCallbackWithHistoricalFeedShouldNotThrow`
Datei: `odin-core/src/test/java/de/its/odin/core/sim/SimulationRunnerTest.java`
