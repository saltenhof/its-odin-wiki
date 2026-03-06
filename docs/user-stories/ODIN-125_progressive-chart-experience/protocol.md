# Protokoll: ODIN-125 — Progressive Chart Experience

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

---

## Design-Entscheidungen

| Entscheidung | Begruendung |
|---|---|
| `backtest-bar` → `chart.appendBar()` via `handleChartEventRef` (imperativer Pfad) | React State fuer 5-20 bars/sec wuerde Render-Storm verursachen. Chart-Library (Lightweight Charts) ist fuer imperative Updates ausgelegt. |
| `simTime` als Top-Level-String in Store | Einfach fuer Store-Updates; kein verschachteltes Objekt noetig da simTime immer ein einziger ISO-8601-String ist. |
| Feldnamen `openTime/closeTime` statt `barOpenTime/barCloseTime` | Konsistenz mit bestehendem Frontend-Code (`BacktestBarProgressPayload` nutzt bereits `currentDate`). |
| Kein explizites `barIndex`-Feld im Payload | Chart nutzt Timestamp-basierte X-Achse. `openTime`-basierter Unix-Timestamp ist genuegend eindeutig. |
| NaN-Guard in `BacktestLiveView.handleChartEventRef` | Schutz am Uebergangspunkt zur Chart-Library. Gemini D2 hat Verschiebung nach `useBacktestStream` vorgeschlagen — bewertet als low priority, da Backend ISO-8601 UTC garantiert und der Guard nur als Defense-in-Depth dient. |
| `useChartData` Mode-Guard (`backtest-live` skippt REST-Fetch) | Klare Trennung: Live-Modus startet leer, alle Bars kommen per SSE. Verhindert Mischung von REST-Daten und SSE-Daten. |
| Auto-Redirect mit `replace: true` | Verhindert History-Pollution. Doppelter Navigate (React StrictMode) ist harmlos durch `replace: true`. |
| `detailLoading`-Flicker-Fix via `isInitialLoad`-Guard | Polling-Updates setzen `detailLoading` nicht mehr auf `true`, nur initiales Laden. Verhindert Flicker-Effekt im UI waehrend laufendem Backtest. |

---

## Offene Punkte

| # | Thema | Empfehlung |
|---|---|---|
| 1 | Stream-Gap Recovery fuer BacktestLiveView | Eigene User Story: `onReconnect`-Callback + Gap-Fill-Logik via REST |
| 2 | IntraDayChart Performance-Refactoring | Eigene Story: `setLiveDataVersion` Batching, `updateActiveBar` O/H/L/C-Fix |
| 3 | Day-Boundary Chart-Reset | Validieren bei erstem echtem Backtest-Lauf mit Multi-Day-Daten |
| 4 | Performance-Monitoring | Metriken fuer Bar-Append-Rate und Render-Latenz bei Produktion-Backtests |
| 5 | NaN-Guard Verschiebung | Gemini D2: Guard koennte in `useBacktestStream` verschoben werden. Low priority solange Backend ISO-8601 UTC garantiert. |

---

## ChatGPT-Sparring

**Prompt-Fokus**: Test-Edge-Cases fuer die `backtest-bar` → `appendBar()` Kette, die Unit-Tests nicht abdecken.

**Ergebnis**: ChatGPT hat 5 hochwertige Integration-Test-Szenarien identifiziert:

| # | Szenario | Bewertung | Massnahme |
|---|---|---|---|
| 1 | Stream-Gap Recovery — BacktestLiveView uebergibt kein `onReconnect` | VALID | Out-of-Scope: Eigene Story fuer Reconnect-Handling erforderlich |
| 2 | Day-Rollover Interleaving — Bars von Tag N+1 kommen waehrend Tag-N-Summary | LOW | Reihenfolge vom Backend garantiert; Client-seitig unkritisch |
| 3 | 10.001-Bar Eviction — Memory-Cap bei MAX_LIVE_BARS | VALID | Abgedeckt durch Integration-Test TC-5 (Memory Cap + FIFO-Eviction) |
| 4 | Late Events nach Reconnect — doppelte/alte Bars | MEDIUM | Backend sendet `lastEventId`; Out-of-Scope fuer ODIN-125 |
| 5 | Redirect Race Condition — Detail-Page Redirect bei schnellem Status-Wechsel | LOW | `replace: true` verhindert History-Pollution; kurze Transition akzeptabel |

**ChatGPT-Empfehlung umgesetzt**: Integration-Tests fuer Aggregations-Boundary-Crossing, NaN-Guard + Recovery, Memory-Cap, simTime-Ordering-Invariante, und Store-Reset (6 Test-Cases in `BacktestLiveView.integration.test.ts`).

---

## Gemini-Review

### Dimension 1: Code-Review

| Schwere | Befund | Bewertung |
|---|---|---|
| HIGH | `setSimTime` auf jedem Bar triggert Render-Storm bei hoher Event-Rate | Pre-existing Design in Zustand (Subscriber-Isolation). `SimClockStrip` isoliert simTime-Reads vom Main-Page-Component. Kein akutes Risiko bei 5-20 bars/sec. |
| HIGH | `setLiveDataVersion` Performance bei hoher Event-Rate | Pre-existing Design in `useChartData`; betrifft den gesamten IntraDayChart, nicht spezifisch ODIN-125. Eigene Refactoring-Story empfohlen. |
| HIGH | O(N) Array-Spreading in `liveBarsRef` (spread + slice) | Pre-existing Pattern in `useChartData`. Bei 10k Bars und 20 bars/sec: 200k Ops/sec — noch akzeptabel. Refactoring auf mutable Ring-Buffer empfohlen fuer Praxis-Optimierung. |
| MEDIUM | NaN-Guard-Platzierung in Presentation Layer | Design-Entscheidung begruendet akzeptiert (s. Design-Entscheidungen). |
| MAJOR | Redirect Double-Fire bei React StrictMode | Akzeptiert: `replace: true` + idempotenter Navigate macht Double-Fire harmlos |
| MINOR | `openTime`-Parsing ohne explizite UTC-Erzwingung | NaN-Guard faengt invalide Timestamps ab; Backend sendet ISO-8601 UTC mit `Z`-Suffix. |

### Dimension 2: Konzepttreue

| Punkt | Bewertung | Details |
|---|---|---|
| Empty Chart Start | CORRECT | `useChartData` Mode-Guard (`backtest-live`) initialisiert leeren Chart ohne REST-Fetch. `liveBarsRef.current = []` korrekt gesetzt. |
| Progressive Rendering | CORRECT | SSE→Router→`chart.appendBar()` Chain ist semantisch korrekt fuer "one bar at a time" Progressive Rendering. |
| SimTime Semantics | CORRECT | `payload.simTime` repraesentiert den internen Clock-Stand des Backtest-Engines (nicht Boundary-Zeitpunkt). Korrekte Wahl. |
| NaN Guard Placement | DEVIATION | In `BacktestLiveView` statt `useBacktestStream`. Akzeptiert als Defense-in-Depth (s. Design-Entscheidungen). |
| Chart Ref Pattern | CORRECT | `useRef<IntraDayChartRef>` + imperative `chart.appendBar()` ist die richtige Wahl fuer High-Frequency Updates ohne React Render-Storm. |

### Dimension 3: Praxis-Risiken

| Risiko | Schwere | In Scope | Bewertung | Massnahme |
|---|---|---|---|---|
| React StrictMode Double-Firing — doppelte Bars durch zwei simultane SSE-Verbindungen | MEDIUM | JA | Grace-Period + single-owner Pattern verhindert doppelte Connections. StrictMode ist nur Development. | Kein ODIN-125-Fix noetig; bestehender Pattern genuegt. |
| Rapid backtestId Navigation — Bars von Run A in Run B Chart | HIGH | NEIN | `backtestId` in `BacktestBarPayload` vorhanden; Client-seitiges Drop moglich, aber Backend-Seitig kontrolliert durch Stream-Teardown. | Out-of-Scope. Eigene Story fuer sichere Navigation-Teardown. |
| Browser Tab Backgrounding — SSE Connection Drop | MEDIUM | NEIN | `EventSource` reconnects automatisch; fehlende Bars koennen nicht via REST nachgeladen werden. | Out-of-Scope. Reconnect-Handling ist eigene Story. |
| Memory ueber volle Session (88k evicted Bars) | HIGH | JA | `liveBarsRef` und Chart-Library-Series sind getrennte Datenhaltungen. Eviction aus `liveBarsRef` betrifft nicht die intern vom Chart gehaltenen Daten. Test TC-5 abgedeckt. | Architektur-Klarstellung: Chart-Library ist Source of Truth fuer Rendering; `liveBarsRef` nur fuer Re-Aggregation. |
| Date Parsing Locale-Sensitivity | LOW | JA | Backend sendet immer `Z`-Suffix (UTC). `isoToUnixSeconds` verwendet `Math.floor(new Date(iso).getTime() / 1000)` — funktioniert korrekt fuer strict ISO-8601 UTC. | Kein Fix noetig. |

---

## Geaenderte Dateien (13 Produktionsdateien + 5 Testdateien + 1 Integrationstestdatei)

| Datei | Aenderung |
|---|---|
| `src/shared/types/sseEvents.ts` | Neues `BacktestBarPayload` Interface, `simTime` auf `BacktestBarProgressPayload`, `backtest-bar` in `SseEvent`-Union |
| `src/shared/types/index.ts` | Barrel-Export fuer `BacktestBarPayload` |
| `src/domains/backtesting/stores/backtestLiveStore.ts` | `simTime`-Feld + `setSimTime`-Action im Zustand-Store |
| `src/domains/backtesting/hooks/useBacktestStream.ts` | `backtest-bar` in `BacktestChartSseEvent`-Union, Routing an Chart-Callback, `setSimTime`-Aufrufe |
| `src/domains/backtesting/pages/BacktestLiveView.tsx` | `backtest-bar`-Case in `handleChartEventRef` mit NaN-Guard, `appendBar()`-Aufruf |
| `src/shared/components/IntraDayChart/hooks/useChartData.ts` | Mode-Guard: `backtest-live` ueberspringt REST-Fetch, Chart startet leer |
| `src/domains/backtesting/pages/BacktestDetailPage.tsx` | Auto-Redirect auf Live-View fuer RUNNING/QUEUED-Backtests |
| `src/domains/backtesting/hooks/useBacktestEvents.ts` | Kontextbezogene Fehlermeldung fuer RUNNING-Status |
| `src/domains/backtesting/stores/backtestingStore.ts` | Fix `detailLoading`-Flicker bei Polling-Updates |
| `src/shared/types/sseEvents.test.ts` | +2 Tests (backtest-bar Parsing, simTime-Feld) |
| `src/domains/backtesting/hooks/useBacktestStream.test.ts` | +3 Tests (backtest-bar Routing, simTime-Propagation, null-simTime) |
| `src/domains/backtesting/stores/backtestLiveStore.test.ts` | +3 Tests (setSimTime, simTime-Reset) |
| `src/domains/backtesting/pages/BacktestLiveView.test.tsx` | +2 Tests (backtest-bar via onChartEvent, NaN-Guard) |
| `src/domains/backtesting/pages/BacktestLiveView.integration.test.ts` | **NEU (Remediation Runde 2)**: 6 Integration-Tests (reale Klassen, kein Mocking) |

---

## Build & Tests

- **Build**: `npm run build` — `built in 3.00s` (tsc + vite, 0 errors)
- **Tests (Runde 1)**: 42 Test-Dateien, **639 Tests bestanden**, 0 fehlgeschlagen
- **Tests (Runde 2)**: 44 Test-Dateien, **661 Tests bestanden**, 0 fehlgeschlagen
- **Neue Tests (Runde 2)**: 6 Integration-Test-Cases in `BacktestLiveView.integration.test.ts`
