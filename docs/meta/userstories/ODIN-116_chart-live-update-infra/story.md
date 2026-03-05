## Kontext

Kern-Infrastruktur fuer Live-Bar-Updates im Chart. Der bestehende `useChartData` Hook laedt Daten einmalig per REST und friert dann ein — es fehlen imperative Update-Methoden fuer Live-Streaming. Ein neuer `useChartLiveUpdates` Hook verarbeitet typisierte Chart-Events zu inkrementellen `series.update()` Aufrufen. Dieser Hook ist SSE-agnostisch und bildet die Bruecke zwischen Domain-Hooks und der Lightweight Charts API.

## Modul

its-odin-ui — `src/shared/components/IntraDayChart/`

## Abhaengigkeiten

Keine (nutzt bestehende Chart-Infrastruktur)

## Scope

### In Scope
- `useChartData` erweitern: imperative Methoden `appendBar()`, `updateActiveBar()`, `updateIndicators()` im Return-Objekt
- Neuer `useChartLiveUpdates` Hook: nimmt typisierte `ChartUpdate`-Events, fuehrt `series.update()` aus — SSE-agnostisch
- `requestAnimationFrame` Batching mit Batch-Cap (max 50 Updates/Frame) und per-Frame Coalescing nach `(seriesId, time)`-Schluessel
- `chartEpoch` State (number, startet bei 0) — wird bei Reconnect inkrementiert, dient als Gating fuer veraltete Updates
- `pendingAggregationBuffer` fuer Multi-Timeframe Live-Updates (1m → 3m/5m/10m inkrementelle Aggregation)
- `ChartUpdate` TypeScript Interface als Eingabetyp fuer den Hook

### Out of Scope
- SSE-Client und SSE-Subscription (ODIN-115)
- DecisionFeed (ODIN-118)
- IntraDayChart mode=backtest-live (ODIN-117)

## Akzeptanzkriterien
- [ ] `useChartData` gibt `appendBar()`, `updateActiveBar()`, `updateIndicators()` zurueck
- [ ] `useChartLiveUpdates` existiert und verarbeitet `ChartUpdate`-Events zu `series.update()` Aufrufen
- [ ] rAF-Batching ist implementiert: Updates werden pro Animation-Frame gebuendelt
- [ ] Batch-Cap von 50 Updates/Frame ist implementiert, ueberschuessige Updates werden ins naechste Frame verschoben
- [ ] Per-Frame Coalescing: Bei mehreren Updates fuer denselben `(seriesId, time)`-Schluessel wird nur der letzte verarbeitet
- [ ] `chartEpoch` State existiert und wird bei `setData()` Aufrufen inkrementiert
- [ ] `pendingAggregationBuffer` aggregiert 1m-Bars korrekt auf 3m/5m/10m Timeframes
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer rAF-Batching, Coalescing und Aggregationslogik existieren

## Technische Details

### Geaenderte Dateien
- `src/shared/components/IntraDayChart/hooks/useChartData.ts` — Erweiterung des Return-Typs:
  ```typescript
  export interface UseChartDataResult {
    // Bestehend:
    readonly data: ChartDayData | null;
    readonly loadingState: ChartDataLoadingState;
    readonly error: string | null;
    readonly reload: () => void;
    // NEU:
    readonly appendBar: (bar: OhlcvBar) => void;
    readonly updateActiveBar: (bar: OhlcvBar) => void;
    readonly updateIndicators: (indicator: IndicatorSnapshot) => void;
    readonly appendTradeEvent: (event: ChartTradeEvent) => void;
  }
  ```

### Neue Dateien
- `src/shared/components/IntraDayChart/hooks/useChartLiveUpdates.ts`:
  ```typescript
  interface ChartUpdate {
    readonly seriesId: string;
    readonly time: number;
    readonly value: Record<string, number>;
  }

  export function useChartLiveUpdates(
    seriesRefs: ReadonlyMap<string, ISeriesApi<SeriesType>>,
    chartEpoch: number,
  ): {
    readonly enqueueUpdate: (update: ChartUpdate) => void;
    readonly flush: () => void;
  };
  ```

- `src/shared/components/IntraDayChart/utils/liveAggregation.ts` — Inkrementelle Aggregationslogik fuer Multi-Timeframe:
  ```typescript
  export function aggregateIncrementalBar(
    buffer: AggregationBuffer,
    newBar: OhlcvBar,
    timeframeMinutes: number,
  ): { readonly aggregatedBar: OhlcvBar; readonly isComplete: boolean };
  ```

### Patterns
- **rAF-Batching:** `pendingUpdatesRef` sammelt Updates, `requestAnimationFrame` wird einmal pro Batch gescheduled, `processBatch()` verarbeitet max 50 und verschiebt den Rest
- **Coalescing:** `Map<string, ChartUpdate>` mit Key `${seriesId}:${time}` — bei Duplikaten gewinnt der letzte
- **chartEpoch:** Simple Zahl, wird bei Reconnect-Recovery hochgezaehlt. Updates mit veraltetem Epoch werden verworfen
- **Aggregation:** Buffer haelt die 1m-Bars der aktuellen Aggregationsgruppe. Bei voller Gruppe (z.B. 5 Bars fuer 5m) wird finalisiert

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §4.4: Erweiterung useChartData, §5.1-§5.2: Bar-Update-Logik, §5.3: Indicator-Lag-Hinweis (beabsichtigt), §5.4: Multi-Timeframe-Aggregation, §5.5: rAF-Batching mit Batch-Cap und Coalescing

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §4: TypeScript strict, readonly, explizite Return-Types, §5: State Management (kein redundanter State), §7: Komponenten-Regeln
- `CLAUDE.md` — Frontend Architecture: keine Business-Logik in Komponenten, Hooks fuer Logik

## Notizen fuer den Implementierer
- `useChartLiveUpdates` darf KEINEN eigenen SSE-Client erstellen — er ist SSE-agnostisch. Die Verdrahtung erfolgt durch den Domain-Hook (usePipelineStream bzw. useBacktestStream).
- Die `series.update()` API von TradingView Lightweight Charts akzeptiert ein Bar-Objekt mit `time`-Feld. Wenn `time` dem letzten bekannten Bar entspricht, wird dieser aktualisiert; wenn `time` neuer ist, wird ein neuer Bar angehaengt. Das ist die Kernmechanik.
- Indikator-Updates im Live-Trading erfolgen NUR fuer abgeschlossene Bars — der aktive Bar hat keine Indikatoren (bewusste Designentscheidung, Concept 12 §5.3).
- Bei Cleanup (Hook-Unmount) muss der `cancelAnimationFrame` aufgerufen werden, um Memory Leaks zu vermeiden.
- Die `pendingAggregationBuffer` muss bei Timeframe-Wechsel zurueckgesetzt werden.

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.ts`) fuer rAF-Batching, Coalescing, Aggregation, chartEpoch-Gating
- [ ] Integrationstests fuer useChartData-Erweiterung mit gemockten Series-Refs
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen exportierten Hooks, Interfaces und Funktionen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
