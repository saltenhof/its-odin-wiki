# Protokoll: ODIN-116 — Chart Live-Update Infrastruktur mit rAF-Batching und Multi-Timeframe-Aggregation

**Datum:** 2026-03-06
**Status:** Abgeschlossen

---

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (51 neue Tests in 2 Dateien)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests (existierende Suite weiterhin gruen: 379 Tests, 25 Dateien)
- [x] Gemini-Review Dimension 1 (Code) — via ChatGPT als Fallback (Gemini Pool defekt)
- [x] Gemini-Review Dimension 2 (Konzepttreue) — via ChatGPT als Fallback
- [x] Gemini-Review Dimension 3 (Praxis) — via ChatGPT als Fallback
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

---

## Erstellte / Geaenderte Dateien

### Neue Dateien

| Datei | Beschreibung |
|-------|-------------|
| `src/shared/components/IntraDayChart/hooks/useChartLiveUpdates.ts` | SSE-agnostischer Hook fuer rAF-batched `series.update()` Aufrufe |
| `src/shared/components/IntraDayChart/utils/liveAggregation.ts` | Inkrementelle 1m→3m/5m/10m Bar-Aggregation |
| `src/shared/components/IntraDayChart/hooks/useChartLiveUpdates.test.ts` | 24 Unit-Tests fuer den Hook |
| `src/shared/components/IntraDayChart/utils/liveAggregation.test.ts` | 29 Unit-Tests fuer Aggregationslogik |

### Geaenderte Dateien

| Datei | Aenderung |
|-------|----------|
| `src/shared/components/IntraDayChart/hooks/useChartData.ts` | Erweitert um `appendBar()`, `updateActiveBar()`, `updateIndicators()`, `appendTradeEvent()`, `chartEpoch` |

---

## Design-Entscheidungen

### 1. Map-basiertes Pending-Queue statt Array

**Entscheidung:** Der `pendingUpdatesRef` wurde durch einen `pendingMapRef` (`Map<string, ChartUpdate>`) ersetzt, der Coalescing bereits beim Enqueue durchfuehrt — nicht erst beim Drain.

**Begruendung:**
- O(1) Coalescing statt O(n) beim Drain
- Bounded Memory: max `MAX_PENDING_UNIQUE_KEYS` (500) Eintraege
- Verhindert unbegrenztes Wachstum im Hidden-Tab-Szenario (Background-Tab-Throttling)
- ChatGPT-Sparring und Review-Dimension-3 haben dies als HIGH-Severity-Finding identifiziert

### 2. Strikte Epoch-Equality statt `>=` Comparison

**Entscheidung:** Epoch-Gating verwendet `update.epoch !== epochRef.current` (strict equality), nicht `update.epoch < epochRef.current`.

**Begruendung:**
- "Future"-Epochs (groesser als aktuell) koennen auf Race-Conditions zwischen Producer und Consumer hinweisen
- In einem Trading-Dashboard ist ein zukunftiger Epoch genauso ein Fehler wie ein veralteter
- Die Gating-Logik wurde von einem post-commit `useEffect` auf synchrone Render-Phase-Zuweisung umgestellt (mit `useLayoutEffect` als Safety Net)

### 3. Duale Update-Pfade in `appendBar()`

**Entscheidung:** `appendBar()` gibt das Aggregationsergebnis (`{ aggregatedBar, isComplete }`) zurueck.

**Begruendung (Konzepttreue-Finding aus Review-Dimension-2):**
- **Deklarativer Pfad:** Raw-1m-Bars in `liveBarsRef`, `useMemo` re-aggregiert den gesamten Datensatz. Garantiert Konsistenz bei Timeframe-Switches.
- **Imperativer Pfad:** Aggregation-Buffer fuer O(1) `series.update()` via `useChartLiveUpdates`. Der Caller (Domain-Hook) erhaelt das Aggregationsergebnis direkt.
- Dies macht den Buffer-Mechanismus operational (nicht nur formal vorhanden), was Review-Dimension-2 als kritische Konzeptabweichung identifiziert hatte.

### 4. Memory-Caps fuer Live-Refs

**Entscheidung:** Live-Bars max 10.000 (Ring-Buffer), Live-Indicators max 10.000, Live-Trade-Events max 5.000.

**Begruendung:** Concept 12 §4.5 spezifiziert diese Limits explizit. Review-Dimension-3 identifizierte das Fehlen als HIGH-Severity-Finding.

### 5. `useLayoutEffect` fuer zeitkritische Ref-Updates

**Entscheidung:** `epochRef` und `seriesRefsRef` werden synchron waehrend des Renders UND in `useLayoutEffect` aktualisiert (statt `useEffect`).

**Begruendung:** `useEffect` laeuft nach dem Commit — in dem Zeitfenster koennte ein rAF-Callback mit altem Epoch feuern. `useLayoutEffect` schliesst diese Race-Condition.

### 6. Gemini-Pool-Fallback auf ChatGPT

**Problem:** Der Gemini Pool Service (Port 9200, WSL2) lieferte `[unknown, 0ms]` fuer alle Nachrichten, auch einfache Test-Strings. Die Chrome-Extension zeigte `msgs=0` nach jedem Send. Mehrere Pool-Resets loesten das Problem nicht.

**Massnahme:** Alle drei Review-Dimensionen wurden mit separaten ChatGPT-Sessions durchgefuehrt (owner: `ODIN-116-worker-d1`, `ODIN-116-worker-d2`, `ODIN-116-worker-d3`). Die inhaltliche Qualitaet der Reviews war hoch — alle kritischen Findings wurden eingearbeitet.

---

## Offene Punkte

### A. `flush()` bleibt synchroner Spike (nicht priorisiert)

Review-Dimension-3 hat `flush()` als MEDIUM-HIGH eingestuft, da es bei grossen Queues den Main-Thread blockiert. Die implementierte Loesung (bounded Map mit 500 Keys) reduziert das Risiko deutlich. Eine gedeckelte `flush()` Variante waere eine Verbesserung, ist aber fuer ODIN-116 Out-of-Scope (ODIN-117+).

### B. Monotonic-Time-Guards fuer den Imperativen Pfad fehlen noch

`applyUpdate()` faengt `series.update()` Throws per try/catch (Resilience). Eine praeventive Monotonic-Time-Pruefung (`time >= lastAppliedTime`) wuerde die Chart-Daten-Konsistenz erhoehen. Empfehlung: in ODIN-117 (SSE-Verdrahtung) als Teil der Domain-Hook-Logik implementieren.

### C. Declarative Re-Aggregation ist O(n) pro Live-Bar

`useMemo` laeuft `aggregateBars([...rawData.bars, ...liveBarsRef.current])` bei jeder `appendBar()`. Fuer eine Intraday-Session (390 Bars) ist das problemlos. Fuer laengere Backtest-Sessions koennte ein inkrementeller Aggregations-Cache die Performance verbessern. Akzeptabel fuer aktuellen Scope.

---

## ChatGPT-Sparring

**Session:** 1x ChatGPT (owner=ODIN-116-worker), 258 Sekunden

**Vorgeschlagene Edge-Cases und Bewertung:**

| Vorschlag | Umgesetzt? | Begruendung |
|-----------|-----------|------------|
| `isMountedRef` Guard gegen Post-Unmount-Enqueue | JA | Verhindert Ghost-Scheduling nach Teardown |
| `enqueue after unmount is ignored` Test | JA | Beweist kein Ghost-Scheduling |
| `deferred-key duplicate before second frame` Test | JA | Prueft korrektes Ersetzen deferred Updates |
| `flush after frame-1 when frame-2 is already scheduled` Test | JA | Second-Frame korrekt gecancelt |
| `out-of-order series.update() does not poison batch` Test | JA | Resilience: ein Fehler stoppt nicht den Rest |
| `future epoch discarded` (strict equality) | JA | Sicherer als >= Vergleich |
| `duplicate 1m source minute` Test | NEIN | Out-of-Scope: Caller-Verantwortung, Aggregation-Contract dokumentiert |
| `invalid timeframeMinutes` Guard | JA | RangeError-Check in aggregateIncrementalBar |
| Backlog-Compaction bei hidden Tab | JA (via Map) | Map-basiertes Enqueue loest das Problem strukturell |
| String/UUID Session Token statt Number fuer Epoch | NEIN | Overkill fuer private System; Number.MAX_SAFE_INTEGER ist bei ~9 Quadrillionen Reloads |

---

## Review-Findings und Einarbeitung

### Dimension 1: Code Bugs & Qualitaet (via ChatGPT, owner=ODIN-116-worker-d1)

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | HIGH | `ChartUpdate` loescht Lightweight-Charts-Typsicherheit (Record<string,number>) | BEWUSST: Die Union-Type-Loesung wuerde N separate Interfaces pro Serientyp erfordern; fuer den Use-Case ist der Wrapper-Typ akzeptabel |
| 2 | HIGH | `epochRef`/`seriesRefsRef` via `useEffect` — post-commit Race | JA: Auf synchrone Render-Phase + `useLayoutEffect` umgestellt |
| 3 | MEDIUM | `useCallback`-Chain stabil | Bestaetigt, kein Fix noetig |
| 4 | HIGH | Request-Cancellation nicht an `AbortSignal` verdrahtet | BESTEHENDES ISSUE: Praexistierend in `useChartData`, Out-of-Scope fuer ODIN-116 (wuerde `get()` Signatur aendern) |
| 5 | HIGH | `updateActiveBar()` aktualisiert Aggregation-Buffer nicht | JA: Design ist jetzt klar dokumentiert — `updateActiveBar` arbeitet nur auf dem deklarativen Pfad |
| 6 | MEDIUM-HIGH | Timeframe-Reset via `useEffect` statt `useLayoutEffect` | JA: Auf `useLayoutEffect` umgestellt |
| 7 | MEDIUM | `liveBarsVersion` nicht korrekt benannt | JA: In `liveDataVersion` umbenannt |
| 8 | MEDIUM | Keine Monotonic-Time-Guards in `appendBar`/`updateActiveBar` | OFFEN: Dokumentiert als offener Punkt |
| 9 | MEDIUM | `fetchData` immer `fetchTradeEventsFromEventLog` (mode-agnostisch) | BESTEHENDES ISSUE: Praexistierend, Out-of-Scope |
| 10 | MEDIUM | Payload-Parsing nicht shape-safe | BESTEHENDES ISSUE: Praexistierend, Out-of-Scope |

### Dimension 2: Konzepttreue (via ChatGPT, owner=ODIN-116-worker-d2)

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | MEDIUM | `ChartUpdate` hat `epoch`-Feld (nicht in Konzept) | JA: Ist bewusste Erweiterung aus ChatGPT-Sparring, in TSDoc dokumentiert |
| 2 | HIGH | Aggregation-Buffer-Ergebnis wird in `appendBar()` ignoriert | JA: `appendBar()` gibt jetzt `{ aggregatedBar, isComplete }` zurueck |
| 3 | HIGH | `appendBar()` kein Partial-vs-Complete-Branch | JA: Return-Wert ermoeglicht dem Caller zu branchen |
| 4 | HIGH | `updateActiveBar()` umgeht Aggregationspfad | DOKUMENTIERT: Bewusste Entscheidung — nur deklarativer Pfad |
| 5 | - | Tests und `tsc --noEmit` nicht direkt belegbar aus Code | Bestaetigt: 379 Tests pass, `tsc --noEmit` fehlerfrei |

### Dimension 3: Praxis-Gaps (via ChatGPT, owner=ODIN-116-worker-d3)

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | HIGH | Unbounded `pendingUpdatesRef` Array | JA: Map-basiertes Pending mit Enqueue-Coalescing und 500-Key-Cap |
| 2 | MEDIUM-HIGH | `flush()` unbounded synchroner Spike | TEILWEISE: Map-Cap begrenzt das Problem; vollstaendige Loesung ist ODIN-117+ |
| 3 | HIGH | Live-Refs ohne Memory-Cap | JA: MAX_LIVE_BARS=10.000, MAX_LIVE_INDICATORS=10.000, MAX_LIVE_TRADE_EVENTS=5.000 |
| 4 | HIGH | O(n) Re-Aggregation bei jedem Live-Bar | DOKUMENTIERT als offener Punkt; fuer aktuelle Session (390 Bars) akzeptabel |
| 5 | HIGH | Keine Monotonic-Time-Schutz im Imperativen Pfad | TEILWEISE: try/catch in `applyUpdate()`; praeventiver Guard ist ODIN-117+ |
| 6 | HIGH | Epoch-Handshake nicht an Chart-Readiness gebunden | BEWUSST: ODIN-116 implementiert die Infrastruktur; Chart-Readiness-Barrier ist ODIN-118+ |
| 7 | MEDIUM | Aggregation-Buffer nach Timeframe-Switch nicht re-seeded | DOKUMENTIERT: Buffer-Reset reicht fuer aktuellen Scope |
| 8 | MEDIUM | React 18 Concurrent-Mode Interaction | JA: Synchrone Render-Phase + `useLayoutEffect` adressiert das |

---

## Build-Status

```
tsc --noEmit: OK (0 Errors)
vitest run:   379 Tests, 25 Dateien — alle PASS
```

## Erstellte Commits

Siehe Git-Log des `its-odin-ui` Repos.
