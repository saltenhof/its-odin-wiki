# QA Report — ODIN-112: BacktestEventPublisher und ThrottlingEventFilter

**Reviewer:** QA Agent (Claude Opus 4.6)
**Datum:** 2026-03-05
**Status:** CONDITIONAL PASS — Findings muessen bearbeitet werden
**Sparring:** ChatGPT (Edge-Case-Analyse), Gemini (3-Dimensionen-Review)

---

## 1. Zusammenfassung

Die Implementierung erfuellt die Akzeptanzkriterien der Story im Kern korrekt. BacktestEventPublisher implementiert MonitoringEventPublisher, remappt Stream-Keys, und ThrottlingEventFilter bietet Three-Tier-Drop-Policy mit Coalescing und Flush-on-Pause. Alle 31 Unit-Tests (20 ThrottlingEventFilter + 11 BacktestEventPublisher) bestehen. Build kompiliert fehlerfrei.

Allerdings wurden durch Multi-LLM-Sparring mehrere Findings identifiziert, davon 3 HIGH-Severity, die vor Production-Einsatz adressiert werden muessen.

---

## 2. Akzeptanzkriterien-Pruefung

| # | Kriterium | Status | Nachweis |
|---|-----------|--------|----------|
| 1 | BacktestEventPublisher implementiert MonitoringEventPublisher Port | PASS | `implements MonitoringEventPublisher` in Klasse |
| 2 | Stream-Key Remapping: instrumentId -> "backtest-{backtestId}" | PASS | `STREAM_KEY_PREFIX + backtestId`, Test `streamKeyShouldBePrefixedWithBacktest` |
| 3 | ThrottlingEventFilter droppt Events korrekt nach konfiguriertem Intervall | PASS | Tests `secondThrottledEventWithinIntervalShouldBeDropped`, `droppableEventsShouldBeThrottled` |
| 4 | NON_DROPPABLE Events werden nie gedroppt | PASS | 5 Tests fuer alle NON_DROPPABLE-Typen |
| 5 | Coalescing: Bei mehreren indicator-updates innerhalb eines Intervalls wird nur der letzte gesendet | PASS | Test `coalescingShouldReturnLatestPayloadAfterInterval`, `flushShouldReturnAllCachedValues` |
| 6 | Flush-on-Pause: Bei Day-Completion werden alle gecachten Werte sofort gesendet | PASS | Tests `flushShouldReturnAllCachedValues`, `flushShouldSendCachedEventsToStream` |
| 7 | Unit-Tests fuer alle Throttling-Szenarien | PASS | 20 Tests in ThrottlingEventFilterTest |
| 8 | Unit-Tests fuer Coalescing | PASS | Coalescing-Tests vorhanden |
| 9 | Build kompiliert fehlerfrei | PASS | `BUILD SUCCESS` |

---

## 3. Findings

### F-01 [HIGH] Race Condition: Non-atomic check-then-act in filter()

**Quelle:** ChatGPT #1, #2, #5 + Gemini Dimension 1 (CRITICAL)

Die `filter()`-Methode fuehrt eine nicht-atomare Sequenz aus:
1. `coalescingCache.put(eventType, payload)`
2. `lastSent.get(eventType)` — Zeitcheck
3. `lastSent.put(eventType, now)`
4. `coalescingCache.remove(eventType)` — gibt den Wert zurueck

Bei konkurrenten Aufrufen (mehrere Worker-Threads im Backtest) kann:
- Thread A und Thread B gleichzeitig den Zeitcheck bestehen
- Einer der `coalescingCache.remove()` Aufrufe `null` zurueckerhalten
- `lastSent` wird trotzdem aktualisiert (Phantom-Send), was nachfolgende Events supprimiert

Zusaetzlich kann `flush()` zwischen Schritt 1 und Schritt 4 den Wert aus dem Cache stehlen.

**Impact:** Event-Verlust unter Konkurrenz. Besonders kritisch bei Flush-on-Pause, wo der finale Zustand verloren gehen kann.

**Empfehlung:** `ConcurrentHashMap.compute()` verwenden, um Zeitcheck und Cache-Extraktion atomar auszufuehren, oder die gesamte filter()-Methode mit einem Lock schuetzen.

---

### F-02 [HIGH] Coalescing-Key nur eventType — Multi-Instrument-Datenverlust

**Quelle:** ChatGPT #3

Der Coalescing-Cache verwendet `eventType` als Schluessel: `coalescingCache.put(eventType, payload)`. Bei Multi-Instrument-Backtests (z.B. AAPL + MSFT) ueberschreibt ein `indicator-update` fuer MSFT den gecachten Wert fuer AAPL.

Da alle Events auf denselben `backtest-{id}` Stream gemappt werden, gehen Instrument-spezifische Snapshot-Werte verloren.

**Impact:** Bei Multi-Instrument-Backtests zeigt die UI nach einem Flush nur den letzten Snapshot eines Instruments, nicht aller Instrumente.

**Empfehlung:** Coalescing-Key auf `eventType + ":" + instrumentId` erweitern. Erfordert, dass `filter()` den instrumentId als zusaetzlichen Parameter erhaelt, oder dass der Payload die Instrument-Identifikation traegt.

**Anmerkung:** Falls ODIN aktuell nur Single-Instrument-Backtests unterstuetzt, ist dies LOW-Priority, muss aber als Known Limitation dokumentiert werden.

---

### F-03 [HIGH] Lifecycle-Events koennen vor finalem Flush ankommen

**Quelle:** ChatGPT #9

Wenn der Aufrufer `publishInstrumentEvent("AAPL", "completed", payload)` VOR `shutdown()`/`flushPendingEvents()` aufruft, empfaengt der Client das `completed`-Event bevor die letzten gecachten Snapshot-Werte geflusht werden. Die UI koennte den Stream schliessen, ohne den finalen Zustand zu zeigen.

**Impact:** Stale UI-State bei Backtest-Completion.

**Empfehlung:** Entweder:
- (a) In `publishInstrumentEvent()` bei Lifecycle-Events automatisch `flushPendingEvents()` VOR dem Senden aufrufen, oder
- (b) In der Dokumentation klar festlegen, dass `shutdown()` vor dem Lifecycle-Event aufgerufen werden muss (und dies durch einen Test absichern).

---

### F-04 [MEDIUM] flush() setzt lastSent auf now — Post-Flush-Verzoegerung

**Quelle:** ChatGPT #6

`flush()` setzt `lastSent.put(eventType, System.nanoTime())` fuer jeden geflushed Key. Dadurch werden Events nach dem Flush fuer ein volles Intervall unterdrueckt. Wenn flush() bei Tag-Wechsel aufgerufen wird, sieht die UI fuer den naechsten Tag keine Updates bis das Intervall abgelaufen ist.

**Impact:** Verzoegerte erste Updates nach Tag-Wechsel.

**Empfehlung:** Nach dem Flush `lastSent` fuer die geflushed Keys entfernen (nicht auf `now` setzen), damit der erste Event des naechsten Tages sofort durchkommt.

---

### F-05 [MEDIUM] Kein Shutdown-Guard — Publisher akzeptiert Events nach shutdown()

**Quelle:** ChatGPT #10

`shutdown()` ruft `flushPendingEvents()` und `filter.reset()` auf, setzt aber kein `closed`-Flag. Nachfolgende `publish*()`-Aufrufe durch spaete Worker-Threads werden akzeptiert und an den EmitterManager weitergeleitet — auf einem moeglicherweise bereits geschlossenen Stream.

**Empfehlung:** `volatile boolean closed`-Flag einfuehren. Nach `shutdown()` sollten `publish*()`-Aufrufe per Guard ignoriert oder geloggt werden.

---

### F-06 [MEDIUM] THROTTLE vs. DROPPABLE nicht unterschieden

**Quelle:** Gemini Dimension 2 (MAJOR)

Das Konzept (§6.4) beschreibt DROPPABLE als "aggressiveres Throttling" gegenueber THROTTLE. Die Implementierung behandelt beide Tiers identisch — der einzige Unterschied liegt in den konfigurierten Intervallen (DROPPABLE hat laengere Intervalle: 2000ms vs. 200-500ms).

**Empfehlung:** Klaeren, ob die unterschiedlichen Intervalle ausreichen (dann dokumentieren) oder ob DROPPABLE ein eigenes Verhalten braucht (z.B. kein Coalescing, pure Drop-Semantik).

---

### F-07 [MINOR] cycle-update verwendet positionUpdateMs statt eigener Konfiguration

**Quelle:** ChatGPT #12, Gemini Dimension 1 (MINOR)

Im ThrottlingEventFilter-Konstruktor:
```java
throttleIntervalsNanos.put(SseEventType.CYCLE_UPDATE.value(),
        TimeUnit.MILLISECONDS.toNanos(throttleProperties.positionUpdateMs()));
```

`CYCLE_UPDATE` teilt sich das Intervall mit `POSITION_UPDATE`. Eigenstaendiges Tuning ist nicht moeglich.

**Empfehlung:** Entweder eigenes Property `cycleUpdateMs` in `BacktestThrottleProperties` aufnehmen oder die Kopplung dokumentieren.

---

### F-08 [MINOR] Zusaetzliche NON_DROPPABLE-Types gegenueber Konzept

**Quelle:** Gemini Dimension 2 (MINOR)

Die Implementierung fuegt `ALERT` und `DEGRADATION_CHANGE` zu den NON_DROPPABLE-Types hinzu. Das Konzept §6.4 listet nur: trade-executed, pipeline-state, completed, failed, cancelled, backtest-day-summary.

**Bewertung:** Die Erweiterung ist fachlich sinnvoll (Alerts und Degradation-Changes sind kritische System-Events), aber eine Konzept-Abweichung, die dokumentiert werden sollte.

---

### F-09 [MINOR] Konfigurierte Default-Intervalle weichen vom Konzept ab

**Quelle:** Eigenanalyse

| Event-Typ | Konzept §6.4 | application.properties | Story |
|-----------|-------------|----------------------|-------|
| indicator-update | 100ms | 200ms | 200ms |
| position-update | 200ms | 500ms | 500ms |
| bar-progress | 500ms | 1000ms | 1000ms |
| quant-score | 250ms | 2000ms | 2000ms |
| gate-result | 250ms | 2000ms | 2000ms |

Die Story-Werte weichen vom Konzept ab. Die Implementierung folgt den Story-Werten. Klaerung erforderlich, welche Werte gelten.

---

### F-10 [INFO] Alle Tests sind single-threaded

**Quelle:** ChatGPT (uebereinstimmend), Gemini

Kein einziger Test prueft konkurrentes Verhalten. Angesichts der Thread-Safety-Anforderungen (ConcurrentHashMap, Worker-Threads im Backtest) sollten zumindest grundlegende Concurrency-Tests hinzugefuegt werden.

---

## 4. Test-Abdeckung

| Testkategorie | Vorhanden | Bewertung |
|---------------|-----------|-----------|
| NON_DROPPABLE pass-through | 5 Tests | Gut |
| THROTTLE-Verhalten | 5 Tests | Gut |
| DROPPABLE-Verhalten | 2 Tests | Ausreichend |
| Coalescing | 2 Tests | Ausreichend |
| Flush-Semantik | 3 Tests | Gut |
| Drop-Counting | 2 Tests | Ausreichend |
| Reset | 1 Test | Minimal |
| Stream-Key-Remapping | 2 Tests | Gut |
| Lifecycle-Bypass | 3 Tests | Gut |
| Flush via Publisher | 2 Tests | Gut |
| Concurrency | 0 Tests | FEHLEND |
| Post-Shutdown-Guard | 0 Tests | FEHLEND |
| Multi-Instrument-Coalescing | 0 Tests | FEHLEND |

---

## 5. Empfohlene Massnahmen

### Vor Merge (Blocking)
1. **F-01:** filter() atomar machen (compute-basiert oder Lock)
2. **F-03:** Flush vor Lifecycle-Events sicherstellen (automatisch oder dokumentiert + getestet)
3. **F-05:** Shutdown-Guard einfuehren

### Vor Production (Should-Fix)
4. **F-02:** Coalescing-Key erweitern oder als Known Limitation dokumentieren
5. **F-04:** lastSent nach Flush entfernen statt auf now setzen
6. **F-10:** Mindestens 2-3 Concurrency-Tests hinzufuegen

### Optional (Nice-to-Have)
7. **F-06:** THROTTLE/DROPPABLE-Unterscheidung klaeren
8. **F-07:** Eigenes cycleUpdateMs-Property
9. **F-08:** NON_DROPPABLE-Erweiterung im Konzept nachfuehren
10. **F-09:** Throttle-Intervalle mit Product Owner abstimmen

---

## 6. Gesamtbewertung

Die Implementierung ist funktional korrekt und erfuellt die Story-Scope-Definition. Code-Qualitaet ist hoch: saubere Benennung, JavaDoc vollstaendig, explizite Typen, keine Magic Numbers. Die Architektur (Adapter-Pattern, Port-Interface) entspricht dem Konzept.

Die identifizierten Concurrency-Probleme (F-01) sind fuer den aktuellen Single-Thread-Testbetrieb nicht sichtbar, werden aber unter Production-Last (84.000 Events, Worker-Threads) auftreten. F-03 und F-05 sind Lifecycle-Korrektheitsprobleme, die beim Wiring in ODIN-114 relevant werden.

**Verdict: CONDITIONAL PASS** — F-01, F-03, F-05 muessen vor Merge in main adressiert werden.
