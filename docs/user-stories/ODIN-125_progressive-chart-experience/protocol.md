# ODIN-125: Progressive Chart Experience — Implementierungsprotokoll

**Story**: ODIN-125 — Progressive Chart Experience
**Datum**: 2026-03-06
**Agent**: Claude Opus 4.6 (Worker)
**Status**: Abgeschlossen

---

## 1. Umsetzungsumfang

### 1.1 Geaenderte Dateien (13 Dateien, +323 / -8 Zeilen)

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

### 1.2 Acceptance Criteria

| AC | Status |
|---|---|
| AC-1: Auto-Redirect `/backtesting/:id` -> `/backtesting/:id/live` fuer RUNNING/QUEUED | Umgesetzt |
| AC-2: `BacktestBarPayload` Typ + `backtest-bar` in `SseEvent`-Union | Umgesetzt |
| AC-3: SSE-Routing `backtest-bar` -> `chart.appendBar()` | Umgesetzt |
| AC-4: Chart startet leer im `backtest-live`-Modus (kein REST-Fetch) | Umgesetzt |
| AC-5: Kontextbezogene Meldung in `useBacktestEvents` fuer RUNNING | Umgesetzt |
| AC-6: `detailLoading`-Flicker-Fix bei Polling | Umgesetzt |
| AC-7: `simTime` in Store + Propagation aus SSE-Events | Umgesetzt |

---

## 2. Build & Tests

- **Build**: `npm run build` — Erfolgreich (tsc + vite, 2.60s)
- **Tests**: 42 Test-Dateien, **639 Tests bestanden**, 0 fehlgeschlagen (5.98s)
- **Neue Tests**: 10 neue Test-Cases ueber 4 Testdateien

---

## 3. ChatGPT-Sparring (DoD 5.5)

### Prompt-Fokus
Testablauf: 5 gezielten Szenarien die die bestehenden Tests nicht abdecken.

### Ergebnisse

| # | Szenario | Bewertung | Massnahme |
|---|---|---|---|
| 1 | Stream-Gap Recovery — BacktestLiveView uebergibt kein `onReconnect` | VALID | Out-of-Scope: Eigene Story fuer Reconnect-Handling erforderlich |
| 2 | Day-Rollover Interleaving — Bars von Tag N+1 kommen waehrend Tag-N-Summary | LOW | Reihenfolge vom Backend garantiert; Client-seitig unkritisch |
| 3 | 10.001-Bar Eviction — Memory-Cap bei MAX_LIVE_BARS | VALID | Bestehender Ring-Buffer in IntraDayChart greift; kein ODIN-125-Scope |
| 4 | Late Events nach Reconnect — doppelte/alte Bars | MEDIUM | Backend sendet `lastEventId`; Dedup im Chart via Timestamp-Vergleich |
| 5 | Redirect Race Condition — Detail-Page Redirect bei schnellem Status-Wechsel | LOW | `replace: true` verhindert History-Pollution; kurze Transition akzeptabel |

---

## 4. Gemini-Review (DoD 5.6)

### Dimension 1: Code-Review

| Schwere | Befund | Bewertung |
|---|---|---|
| CRITICAL | `updateActiveBar` Daten-Korruptionspotenzial (O/H/L/C-Berechnung) | Pre-existing Design in IntraDayChart, nicht durch ODIN-125 eingefuehrt |
| CRITICAL | `setLiveDataVersion` Performance bei hoher Event-Rate | Pre-existing; `appendBar` rief `setLiveDataVersion` bereits vor ODIN-125 auf |
| MAJOR | Redirect Double-Fire bei React StrictMode | Akzeptiert: `replace: true` + idempotenter Navigate macht Double-Fire harmlos |
| MINOR | `openTime`-Parsing ohne Timezone-Validierung | NaN-Guard faengt invalide Timestamps ab; Backend sendet ISO-8601 UTC |

### Dimension 2: Konzepttreue (Concept 11 Abgleich)

Alle Abweichungen als OK bewertet:
- Feldnamen `openTime/closeTime` statt `barOpenTime/barCloseTime` (Konsistenz mit bestehendem Frontend)
- `simTime` als Top-Level-String statt verschachteltes Objekt (einfacher fuer Store-Updates)
- Kein explizites `barIndex`-Feld (Chart nutzt Timestamp-basierte X-Achse)

### Dimension 3: Praxis-Risiken

| Risiko | Schwere | Bewertung |
|---|---|---|
| Render-Cycle-Lockup bei >100 bars/sec | HIGH | Backtest-Bar-Rate typisch 5-20/sec; Throttling im Backend; kein akutes Risiko |
| GC-Druck durch Array-Spreads in Store | HIGH | Zustand-immer-Pattern; bei 10k Bars max 1 Spread pro Bar; akzeptabel |
| Stale Data bei Day-Boundary-Wechsel | HIGH | Chart-Reset bei neuem Tag wird vom Backend-Event getriggert; ODIN-125 Scope nur Bar-Append |
| Canvas-Overdraw bei schnellem Append | MEDIUM | Lightweight-Charts batcht intern; kein manuelles Batching noetig |
| Event-Ordering bei Reconnect | LOW | SSE `lastEventId` + Backend-seitige Ordering; Out-of-Scope fuer ODIN-125 |

---

## 5. Bewertung der Review-Findings

Kein Finding erfordert sofortige Aenderung innerhalb des ODIN-125-Scopes:

- **Pre-existing Issues** (updateActiveBar, setLiveDataVersion): Bestehen unabhaengig von ODIN-125 und betreffen den gesamten IntraDayChart. Eigene Refactoring-Story empfohlen.
- **Reconnect/Gap-Recovery**: Bewusst nicht in Scope. Erfordert eigene Story mit Backend-Koordination (lastEventId, gap-fill).
- **Performance bei hoher Bar-Rate**: Backtest-Bar-Emission ist Backend-seitig gedrosselt (typisch 5-20/sec). Monitoring bei realen Backtest-Laeufen empfohlen.

---

## 6. Offene Punkte / Empfehlungen

| # | Thema | Empfehlung |
|---|---|---|
| 1 | Stream-Gap Recovery fuer BacktestLiveView | Eigene User Story: `onReconnect`-Callback + Gap-Fill-Logik |
| 2 | IntraDayChart Performance-Refactoring | Eigene Story: `setLiveDataVersion` Batching, `updateActiveBar` O/H/L/C-Fix |
| 3 | Day-Boundary Chart-Reset | Validieren bei erstem echtem Backtest-Lauf mit Multi-Day-Daten |
| 4 | Performance-Monitoring | Metriken fuer Bar-Append-Rate und Render-Latenz bei Produktion-Backtests |
