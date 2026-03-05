## Kontext

Domain-lokaler Zustand und SSE-Hook fuer Backtest-Live-Streaming. Der `backtestLiveStore` haelt den gesamten Backtest-Live-State (Progress, Day-Summaries, Decision-Events, Connection-Status), und der `useBacktestStream` Hook ist der einzige SSE-Client-Owner fuer den Backtest-Stream. Er routet Events deklarativ an den Store und imperativ an den Chart.

## Modul

its-odin-ui — `src/domains/backtesting/`

## Abhaengigkeiten

#8 (ODIN-115: SSE-Types und Backtest-SSE-Client)

## Scope

### In Scope
- `backtestLiveStore` (Zustand): bars, indicators, trades, decisionEvents, progress, daySummaries, connectionStatus, currentDay, detailLevel, backtestStatus
- `useBacktestStream(backtestId, detailLevel)` Hook: einziger SSE-Client-Owner, routet Events an Store + Chart via Callbacks
- Memory-Windowing: `MEMORY_LIMITS` Konstante in `src/shared/config/memoryLimits.ts` (zentral fuer alle Stores)
- Zentrale Memory-Limits: `DECISION_EVENTS_MAX=1000`, `CHART_BARS_MAX=2000`, `TRADE_EVENTS_MAX=500`, `DAY_SUMMARIES_MAX=500`
- FIFO-Eviction bei Ueberschreitung der Limits

### Out of Scope
- UI-Komponenten (ODIN-120, ODIN-121)
- Chart-Integration (ODIN-117)
- Reconnect-Recovery (ODIN-123)

## Akzeptanzkriterien
- [ ] `backtestLiveStore` existiert als Zustand-Store mit allen definierten State-Feldern und Actions
- [ ] `useBacktestStream` Hook erstellt genau einen SSE-Client pro `backtestId`
- [ ] Hook routet `backtest-bar-progress` an `store.updateProgress()`
- [ ] Hook routet `backtest-day-summary` an `store.addDaySummary()`
- [ ] Hook routet `backtest-quant-score`, `backtest-gate-result`, `llm-update`, `trade-update` an `store.addDecisionEvent()`
- [ ] Hook ruft optionalen `onChartEvent` Callback fuer Chart-relevante Events auf
- [ ] `MEMORY_LIMITS` Konstante existiert in `src/shared/config/memoryLimits.ts`
- [ ] Store-Actions enforieren FIFO-Eviction bei Ueberschreitung der Limits
- [ ] SSE-Client wird bei Hook-Unmount korrekt geschlossen
- [ ] SSE-Client wird bei `detailLevel`-Wechsel reconnected (Close + Re-Open)
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer Store-Actions (insb. Eviction) und Hook-Lifecycle existieren

## Technische Details

### Neue Dateien
- `src/shared/config/memoryLimits.ts`:
  ```typescript
  /** Central memory limits for all stores. Prevents unbounded growth in long-running sessions. */
  export const MEMORY_LIMITS = {
    DECISION_EVENTS_MAX: 1_000,
    CHART_BARS_MAX: 2_000,
    TRADE_EVENTS_MAX: 500,
    DAY_SUMMARIES_MAX: 500,
  } as const;
  ```

- `src/domains/backtesting/stores/backtestLiveStore.ts`:
  ```typescript
  interface BacktestLiveState {
    // Progress
    readonly completedDays: number;
    readonly totalDays: number;
    readonly completedBars: number;
    readonly totalBars: number;
    readonly currentDate: string;
    readonly estimatedRemainingMs: number | null;

    // Day Summary
    readonly lastDaySummary: BacktestDaySummaryData | null;
    readonly daySummaries: ReadonlyArray<BacktestDaySummaryData>;

    // Decision Feed Events
    readonly decisionEvents: ReadonlyArray<DecisionFeedEvent>;

    // Detail Level
    readonly detailLevel: 'progress' | 'trading' | 'full';

    // Connection
    readonly connectionStatus: 'connecting' | 'connected' | 'disconnected' | 'error';

    // Backtest Status
    readonly backtestStatus: 'running' | 'paused' | 'completed' | 'failed' | 'cancelled';

    // Actions
    updateProgress: (progress: BacktestBarProgressPayload) => void;
    addDaySummary: (summary: BacktestDaySummaryPayload) => void;
    addDecisionEvent: (event: DecisionFeedEvent) => void;
    setDetailLevel: (level: 'progress' | 'trading' | 'full') => void;
    setConnectionStatus: (status: BacktestLiveState['connectionStatus']) => void;
    setBacktestStatus: (status: BacktestLiveState['backtestStatus']) => void;
    reset: () => void;
  }
  ```

- `src/domains/backtesting/hooks/useBacktestStream.ts`:
  ```typescript
  export function useBacktestStream(
    backtestId: string | undefined,
    detailLevel: 'progress' | 'trading' | 'full',
    onChartEvent?: (event: ChartUpdate) => void,
  ): void;
  ```
  Der Hook:
  1. Erstellt `createBacktestSseClient(backtestId, detailLevel, handleEvent)` bei Mount
  2. Routet Events an den Store (deklarativ) und an `onChartEvent` (imperativ)
  3. Schliesst den Client bei Unmount
  4. Reconnected bei `detailLevel`-Wechsel (Close + Re-Open)

### Event-Routing-Tabelle
| SSE Event-Typ | Store-Action | Chart-Callback |
|---------------|-------------|----------------|
| `backtest-bar-progress` | `updateProgress()` | Nein |
| `backtest-day-summary` | `addDaySummary()` | Nein |
| `backtest-quant-score` | `addDecisionEvent()` | Nein |
| `backtest-gate-result` | `addDecisionEvent()` | Nein |
| `llm-update` | `addDecisionEvent()` | Nein |
| `trade-update` | `addDecisionEvent()` | `onChartEvent` (Trade-Marker) |
| `market-snapshot` | — | `onChartEvent` (Bar-Update) |
| `indicator-update` | — | `onChartEvent` (Indikator-Update) |
| `stream-gap` | — | Nein (ODIN-123 behandelt dies) |

### Patterns
- **Single-Owner:** `useBacktestStream` ist der EINZIGE SSE-Client-Owner. Kein anderer Hook oder Komponente darf einen zweiten Client fuer denselben Stream erstellen.
- **FIFO-Eviction:** `addDecisionEvent` prueft `state.decisionEvents.length >= MEMORY_LIMITS.DECISION_EVENTS_MAX` und entfernt das aelteste Element.
- **Detail-Level-Wechsel:** `useEffect` mit `[detailLevel]` Dependency schiesst den bestehenden Client und oeffnet einen neuen.

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §4.2: backtestLiveStore State-Definition, §4.3: useBacktestStream Hook-Signatur und Event-Routing, §4.5: Memory-Windowing-Strategie mit MEMORY_LIMITS
- `docs/concept/12-live-chart-components.md` — §3.1: SSE-Ownership-Designregel (Single-Owner-Pattern)

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §2: Domain-Stores in domains/, nicht in shared/, §5: State Management (SSE-Hooks als primaere Datenquelle), §4: TypeScript strict, readonly
- `CLAUDE.md` — Frontend Architecture: Union Types statt enum

## Notizen fuer den Implementierer
- Zustand (v5) ist bereits im Projekt installiert — der bestehende `globalStore` und `tradingStore` verwenden es.
- Der Store liegt in `src/domains/backtesting/stores/` — Domain-lokal, da nur Backtest-Komponenten ihn nutzen.
- `MEMORY_LIMITS` liegt in `src/shared/config/` weil es von mehreren Domains genutzt wird (backtestLiveStore UND tradingStore verwenden dasselbe `DECISION_EVENTS_MAX`).
- Bei der FIFO-Eviction: `Array.prototype.slice()` ist ausreichend — kein Ring-Buffer noetig. Bei 1000 Events ist die Performance unkritisch.
- Der `onChartEvent` Callback wird per `useRef` gespeichert, nicht per `useCallback`, um Stale-Closure-Probleme zu vermeiden.
- StrictMode-Handling: React 18 StrictMode ruft Effects doppelt auf. Der Hook muss damit umgehen koennen (Cleanup bei erstem Unmount, dann Re-Mount). Das Pattern aus `usePipelineStream` (100ms Grace) kann uebernommen werden.

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.ts`) fuer Store-Actions (Eviction, Reset, State-Updates)
- [ ] Unit-Tests fuer Hook-Lifecycle (Mount, Unmount, Detail-Level-Wechsel)
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen exportierten Hooks, Stores und Interfaces
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
