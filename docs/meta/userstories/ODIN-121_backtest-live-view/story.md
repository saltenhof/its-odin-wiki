## Kontext

Die Hauptseite fuer Backtest-Live-Monitoring: Chart + DecisionFeed + Progress zusammengebaut. Waehrend eines Backtest-Laufs (30-90 Minuten) soll der Operator in Echtzeit sehen, was die Strategie tut — Kursverlauf, Indikatoren, Trade-Marker, Entscheidungen und Fortschritt. Diese Seite verdrahtet alle zuvor erstellten Komponenten und Hooks zu einer funktionierenden Live-View.

## Modul

its-odin-ui — `src/domains/backtesting/pages/`

## Abhaengigkeiten

#9 (ODIN-116: Chart Live-Update Infrastruktur)
#10 (ODIN-117: IntraDayChart mode=backtest-live)
#11 (ODIN-118: DecisionFeed Component)
#12 (ODIN-119: Backtest-Live Store und Stream-Hook)
#13 (ODIN-120: BacktestProgressBar und DaySummary)

## Scope

### In Scope
- `BacktestLiveView` Page-Komponente: Layout gemaess Wireframe (ProgressBar + DaySummary oben ~80px, Chart Mitte ~55%, DecisionFeed unten ~35%)
- Integration aller Komponenten: IntraDayChart (mode=backtest-live), DecisionFeed, BacktestProgressBar, BacktestDaySummary
- `useBacktestStream` Hook als einziger SSE-Owner — Event-Routing: SSE → Store (deklarativ) + Chart (imperativ)
- `DetailLevelToggle` Komponente: Umschalten zwischen progress/trading/full Detail-Level
- SSE-Reconnect bei Detail-Level-Wechsel (Close + Re-Open via useBacktestStream)
- Route: `/backtests/{backtestId}/live` im AppRouter
- Header mit Backtest-ID, Instrument, SimDate, Status-Badge, Connection-Status

### Out of Scope
- Reconnect-Recovery mit REST-Refresh (ODIN-123)
- Live-Trading-View (ODIN-122)
- Neue Shared-Components (alle muessen bereits existieren)

## Akzeptanzkriterien
- [ ] `BacktestLiveView` rendert das vollstaendige Layout gemaess Wireframe §2.2
- [ ] ProgressBar zeigt aktuellen Fortschritt (Tage, Bars, Restzeit)
- [ ] DaySummary zeigt P&L des letzten abgeschlossenen Tages
- [ ] IntraDayChart zeigt Live-Bars im `backtest-live` Modus
- [ ] DecisionFeed zeigt Decision-Events mit Filterung
- [ ] `useBacktestStream` ist der einzige SSE-Client-Owner auf der Seite
- [ ] Chart erhaelt Bar/Indikator-Updates via `onChartEvent` Callback (imperativ)
- [ ] DecisionFeed liest Events aus `backtestLiveStore` (deklarativ via Selector)
- [ ] `DetailLevelToggle` wechselt zwischen progress/trading/full
- [ ] Detail-Level-Wechsel loest SSE-Reconnect aus
- [ ] Route `/backtests/:backtestId/live` ist registriert
- [ ] `backtestId` wird aus URL-Parametern extrahiert
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Integrationstests fuer Page-Rendering und Event-Routing existieren

## Technische Details

### Neue Dateien
- `src/domains/backtesting/pages/BacktestLiveView.tsx`:
  ```typescript
  /** Main page for live backtest monitoring. Orchestrates Chart, DecisionFeed, Progress and SSE. */
  export function BacktestLiveView(): React.ReactElement;
  ```
  Die Seite:
  1. Extrahiert `backtestId` aus `useParams()`
  2. Verwaltet `detailLevel` State (default: 'trading')
  3. Ruft `useBacktestStream(backtestId, detailLevel, handleChartEvent)` auf
  4. Liest Store-State via Selectors fuer Progress, DaySummary, DecisionEvents
  5. Verdrahtet `handleChartEvent` mit `useChartLiveUpdates.enqueueUpdate`
  6. Rendert Layout

- `src/domains/backtesting/pages/BacktestLiveView.module.css`

- `src/domains/backtesting/components/DetailLevelToggle.tsx`:
  ```typescript
  interface DetailLevelToggleProps {
    readonly activeLevel: 'progress' | 'trading' | 'full';
    readonly onLevelChange: (level: 'progress' | 'trading' | 'full') => void;
  }
  ```
  Drei Buttons/Tabs: Progress (minimal), Trading (Standard), Full (alles). Tooltip erklaert was jeder Level sendet.

- `src/domains/backtesting/components/DetailLevelToggle.module.css`

### Geaenderte Dateien
- `src/app/AppRouter.tsx` (oder equivalenter Router): Neue Route `/backtests/:backtestId/live` → `BacktestLiveView`

### Layout-Struktur
```
BacktestLiveView
├── Header (Backtest-ID, Instrument, SimDate, Status, ConnectionBanner, DetailLevelToggle)
├── BacktestProgressBar (Selector: progress State)
├── BacktestDaySummary (Selector: lastDaySummary)
├── IntraDayChart (mode="backtest-live", onChartEvent)
└── DecisionFeed (events: Selector decisionEvents, Filter-State lokal)
```

### Event-Routing (Single-Owner-Pattern)
```
useBacktestStream (SSE-Owner)
  ├── Store-Pfad (deklarativ): backtest-bar-progress → store.updateProgress()
  │                             backtest-day-summary → store.addDaySummary()
  │                             quant/gate/llm/trade → store.addDecisionEvent()
  └── Chart-Pfad (imperativ):  market-snapshot → handleChartEvent() → enqueueUpdate()
                                indicator-update → handleChartEvent() → enqueueUpdate()
                                trade-update → handleChartEvent() → enqueueUpdate()
```

### Patterns
- **Single-Owner:** `useBacktestStream` ist der EINZIGE SSE-Client. IntraDayChart erstellt KEINEN eigenen Client.
- **Dual-Update-Path:** Store-Updates (React re-render) fuer langsame Daten (Progress, Events). Imperative Updates (series.update) fuer schnelle Daten (Bars, Indikatoren).
- **Detail-Level-State:** `useState` in BacktestLiveView. Wechsel wird an `useBacktestStream` propagiert, der den SSE-Client reconnected.

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §2.2: Backtest-Live-View Wireframe, §3.1: Komponentenbaum, §3.5: Feature-Isolation und Kommunikation, §4.1: Datenfluss SSE→Store→UI, §8: User-Control-Matrix
- `docs/concept/12-live-chart-components.md` — §13: Stories S10 (Page-Integration), S13 (Detail-Level-Toggle)

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §2: Pages in domains/backtesting/pages/, §5: State Management (SSE-Hooks primaer), §7: Komponenten-Regeln
- `CLAUDE.md` — Frontend Architecture: Feature-based folders, keine Feature→Feature Imports

## Notizen fuer den Implementierer
- Diese Story ist eine reine Integrationsarbeit — alle Einzel-Komponenten existieren bereits. Der Wert liegt in der korrekten Verdrahtung.
- Der `handleChartEvent` Callback muss per `useRef` gespeichert werden (nicht `useCallback`), um Stale-Closure-Probleme mit den Chart-Refs zu vermeiden.
- Die Proportionen (80px Header/Progress, 55% Chart, 35% Feed) sollten via CSS Grid oder Flexbox mit festen Hoehen fuer den oberen Bereich und flex-grow fuer Chart/Feed realisiert werden.
- Der `DetailLevelToggle` loest einen SSE-Reconnect aus. Waehrend des Reconnects sollte ein kurzer Loading-State sichtbar sein (ConnectionBanner zeigt "Connecting...").
- `backtestId` muss validiert werden — wenn undefined oder leer, sollte eine Fehlermeldung angezeigt werden (nicht crashen).
- Die DecisionFeed-Events kommen aus dem Store (Zustand Selector). Der Filter-State ist lokal in der BacktestLiveView oder im DecisionFeed selbst.

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.tsx`) fuer BacktestLiveView Rendering und DetailLevelToggle
- [ ] Integrationstests fuer Event-Routing (SSE → Store → UI Update Flow mit gemocktem Stream)
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen exportierten Komponenten
- [ ] CSS Modules mit Dark-Theme-Tokens
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
