## Kontext

Grundlage fuer das gesamte Frontend-Backtest-Streaming: Das Frontend benoetigt neue SSE-Event-Types fuer Backtest-Events sowie eine SSE-Client-Factory fuer den Backtest-Endpoint. Ohne diese Types und den Client koennen keine weiteren Backtest-Live-Features implementiert werden. Zusaetzlich wird das `stream-gap` Control-Event benoetigt, das bei Ring-Buffer-Overflow vom Backend gesendet wird und eine REST-Recovery ausloesen soll.

## Modul

its-odin-ui — `src/shared/types/` und `src/shared/api/`

## Abhaengigkeiten

Keine (Frontend-intern, erste Story im Epic)

## Scope

### In Scope
- 4 neue Backtest-Events als Discriminated Union Members in `sseEvents.ts`: `backtest-bar-progress`, `backtest-day-summary`, `backtest-quant-score`, `backtest-gate-result`
- `stream-gap` Control-Event in `sseEvents.ts` Discriminated Union
- TypeScript Interfaces fuer alle neuen Event-Payloads (`BacktestBarProgressPayload`, `BacktestDaySummaryPayload`, `BacktestQuantScorePayload`, `BacktestGateResultPayload`, `StreamGapPayload`)
- `createBacktestSseClient(backtestId, detailLevel, onEvent)` Factory-Funktion in `sseClient.ts`
- Verbindung zum Endpoint `/api/v1/stream/backtests/{backtestId}?detailLevel={detailLevel}`

### Out of Scope
- Hooks, Stores, UI-Komponenten
- Reconnect-Recovery-Logik (eigene Story ODIN-123)
- Aenderungen an bestehenden Event-Types

## Akzeptanzkriterien
- [ ] `sseEvents.ts` enthaelt 5 neue Discriminated Union Members (4 Backtest-Events + stream-gap)
- [ ] Alle Payload-Interfaces sind typsicher mit `readonly`-Modifiern
- [ ] `createBacktestSseClient` existiert in `sseClient.ts` und akzeptiert `backtestId: string`, `detailLevel: 'progress' | 'trading' | 'full'`, `onEvent: (event: SseEvent) => void`
- [ ] Die Factory nutzt den bestehenden `SseClient`-Wrapper mit Exponential Backoff und Last-Event-ID
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer Event-Type-Guards und Payload-Validierung existieren

## Technische Details

### Neue Dateien
- Keine neuen Dateien — Erweiterung bestehender Dateien

### Geaenderte Dateien
- `src/shared/types/sseEvents.ts` — Erweiterung der `SseEvent` Discriminated Union um 5 neue Members:
  ```typescript
  | { type: 'backtest-bar-progress'; payload: BacktestBarProgressPayload }
  | { type: 'backtest-day-summary'; payload: BacktestDaySummaryPayload }
  | { type: 'backtest-quant-score'; payload: BacktestQuantScorePayload }
  | { type: 'backtest-gate-result'; payload: BacktestGateResultPayload }
  | { type: 'stream-gap'; payload: StreamGapPayload }
  ```
- `src/shared/api/sseClient.ts` — Neue Factory-Funktion `createBacktestSseClient`:
  ```typescript
  export function createBacktestSseClient(
    backtestId: string,
    detailLevel: 'progress' | 'trading' | 'full',
    onEvent: (event: SseEvent) => void,
  ): SseClient;
  ```
  Die Factory konstruiert die URL `/api/v1/stream/backtests/${backtestId}?detailLevel=${detailLevel}` und nutzt den bestehenden `SseClient`-Wrapper.

### Payload-Interfaces (in `sseEvents.ts`)
```typescript
interface BacktestBarProgressPayload {
  readonly backtestId: string;
  readonly completedDays: number;
  readonly totalDays: number;
  readonly completedBars: number;
  readonly totalBars: number;
  readonly currentDate: string;
  readonly estimatedRemainingMs: number | null;
}

interface BacktestDaySummaryPayload {
  readonly backtestId: string;
  readonly tradingDate: string;
  readonly dayPnl: number;
  readonly cumulativePnl: number;
  readonly totalTrades: number;
  readonly winRate: number;
  readonly maxDrawdown: number;
}

interface BacktestQuantScorePayload {
  readonly backtestId: string;
  readonly instrumentId: string;
  readonly timestamp: string;
  readonly overallScore: number;
  readonly intent: string;
  readonly activeSetup: string;
  readonly signals: ReadonlyArray<string>;
}

interface BacktestGateResultPayload {
  readonly backtestId: string;
  readonly instrumentId: string;
  readonly timestamp: string;
  readonly passed: boolean;
  readonly gateResults: ReadonlyArray<{ readonly gate: string; readonly passed: boolean; readonly reason: string }>;
  readonly riskRewardRatio: number | null;
  readonly positionSize: number | null;
}

interface StreamGapPayload {
  readonly reason: string;
  readonly missedEventCount: number;
  readonly lastKnownEventId: string;
}
```

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §1.5: Abhaengigkeit Concept 11, §3.3: Erweiterungen sseEvents.ts und sseClient.ts
- `docs/concept/11-live-data-streaming.md` — §3: Event-Schemas (Backtest-Events), §7.5: Stream-Gap-Signalisierung

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §4: TypeScript-Regeln (strict, Discriminated Unions, readonly), §6: SSE-Client in shared/api/
- `CLAUDE.md` — Frontend Architecture: Discriminated Unions fuer SSE Events, Union Types statt enum

## Notizen fuer den Implementierer
- Die bestehende `SseEvent` Union in `sseEvents.ts` hat aktuell 10 Members. Die 5 neuen Members muessen konsistent mit dem bestehenden Pattern hinzugefuegt werden.
- `createBacktestSseClient` folgt dem Pattern von `createInstrumentSseClient` und `createGlobalSseClient` — gleiche Struktur, anderer Endpoint.
- Der `detailLevel` Parameter wird als Query-Parameter an die URL angehaengt, NICHT als Header.
- `stream-gap` ist kein Backtest-spezifisches Event — es kann auf jedem SSE-Stream auftreten. Daher gehoert es in die allgemeine Union, nicht in eine Backtest-Sub-Union.
- Alle Payload-Interfaces muessen `readonly` verwenden (Frontend-Guardrail §4).

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.ts`) fuer Type-Guards und Payload-Validierung
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen exportierten Types, Interfaces und Funktionen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
