## Kontext

Die bestehende LiveTradingView (aktuell LiveChartPage) zeigt einen eingefrorenen Chart — `useChartData` laedt einmalig per REST und friert dann ein. Kein SSE-Event aktualisiert die Bars. Diese Story erweitert die View um DecisionFeed und korrekte Live-Bar-Streaming-Verdrahtung, sodass der Operator in Echtzeit Kursbewegungen, Indikatoren, Trade-Marker und Pipeline-Entscheidungen sieht.

## Modul

its-odin-ui — `src/domains/trading-operations/`

## Abhaengigkeiten

#9 (ODIN-116: Chart Live-Update Infrastruktur)
#10 (ODIN-117: IntraDayChart mode=backtest-live — fuer ChartMode-Erweiterung und useAutoScroll)
#11 (ODIN-118: DecisionFeed Component)

## Scope

### In Scope
- DecisionFeed in LiveTradingView integrieren (Layout: Chart oben ~65%, DecisionFeed unten ~35%)
- `usePipelineStream` als einziger SSE-Owner (bestehend) — Event-Routing an Chart + DecisionFeed erweitern
- IntraDayChart: mode="live" mit `useChartLiveUpdates` verdrahten (`market-snapshot` → `appendBar`/`updateActiveBar`)
- Trade-Marker: SSE `trade-update` → `TradingEventLayer.addTradeEvent()`
- Indicator-Updates: SSE `indicator-update` → Layer-Updates via `useChartLiveUpdates`
- Decision-Events in `tradingStore.decisionEvents` sammeln (max 1000, FIFO via MEMORY_LIMITS)

### Out of Scope
- Backtest-View (ODIN-121)
- Reconnect-Recovery (ODIN-123)
- Neue SSE-Client-Factories (bestehender `usePipelineStream` reicht)

## Akzeptanzkriterien
- [ ] LiveTradingView zeigt IntraDayChart (mode="live") oben und DecisionFeed unten
- [ ] Chart aktualisiert sich live: `market-snapshot` Events fuehren zu Bar-Updates
- [ ] Neue Bars werden angehaengt wenn die 1m-Grenze ueberschritten wird
- [ ] Aktiver Bar wird inkrementell aktualisiert (OHLCV)
- [ ] Indikatoren werden bei abgeschlossenen Bars aktualisiert
- [ ] Trade-Marker erscheinen in Echtzeit im Chart
- [ ] DecisionFeed zeigt LLM-, Quant-, Gate-, Order- und Alert-Events
- [ ] `usePipelineStream` ist der einzige SSE-Client auf der Seite
- [ ] `tradingStore.decisionEvents` wird mit Pipeline-Entscheidungen befuellt
- [ ] FIFO-Eviction bei >1000 Events funktioniert
- [ ] ControlsPanel (Kill-Switch, Pause/Resume) bleibt integriert (links)
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer Event-Routing und Layout existieren

## Technische Details

### Geaenderte Dateien
- `src/domains/trading-operations/pages/LiveChartPage.tsx` (oder Umbenennung zu `LiveTradingView.tsx`):
  - Layout umbauen: ControlsPanel links (~120px), Chart + DecisionFeed rechts
  - Chart-Bereich: IntraDayChart mit `onChartEvent` Callback
  - DecisionFeed-Bereich: Events aus `tradingStore.decisionEvents` via Selector
  - `handleChartEvent` Callback verdrahten mit `useChartLiveUpdates.enqueueUpdate`

- `src/domains/trading-operations/hooks/usePipelineStream.ts`:
  - Event-Routing erweitern: `market-snapshot` → `onChartEvent` Callback
  - Event-Routing erweitern: `indicator-update` → `onChartEvent` Callback
  - Event-Routing erweitern: `trade-update` → `onChartEvent` Callback + `tradingStore.addDecisionEvent()`
  - Event-Routing erweitern: `llm-update`, `state-change`, `order-update`, `alert` → `tradingStore.addDecisionEvent()`
  - Neuer optionaler `onChartEvent` Parameter in der Hook-Signatur

- `src/domains/trading-operations/stores/tradingStore.ts`:
  - Neues Feld: `decisionEvents: ReadonlyArray<DecisionFeedEvent>` (max 1000 via MEMORY_LIMITS)
  - Neue Action: `addDecisionEvent(event: DecisionFeedEvent)`
  - FIFO-Eviction bei Ueberschreitung

### Layout-Struktur (gemaess Wireframe §2.1)
```
LiveTradingView
├── Header (Instrument, Price, State-Badge, Connection-Status, Controls)
├── ControlsPanel (links, ~120px, bestehend)
└── Main Content (rechts)
    ├── IntraDayChart (mode="live", ~65% Hoehe)
    └── DecisionFeed (~35% Hoehe)
```

### Patterns
- **Single-Owner:** `usePipelineStream` bleibt der einzige SSE-Client. Der Chart bekommt Updates imperativ via Callback.
- **Dual-Update-Path:** Store fuer langsame Daten (DecisionEvents, Alerts), imperative API fuer schnelle Daten (Bars, Indikatoren).
- **Bestehende Infrastruktur:** ControlsPanel, ConnectionBanner, State-Badge sind bereits vorhanden — nur Layout-Integration.

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §2.1: Live-Trading-View Wireframe, §10: Dual-Use (Live + Backtest), §5.2: Bar-Update-Logik (market-snapshot → Bar), §5.3: Indikator-Lag bei Live-Tick, §5.6: Trade-Marker-Live-Update
- `docs/concept/12-live-chart-components.md` — §13: Story S11 (LiveTradingView-Erweiterung)

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §2: Pages in domains/trading-operations/pages/, §5: State Management (SSE-Hooks primaer), §7: Komponenten-Regeln
- `CLAUDE.md` — Frontend Architecture: Feature-based folders, SSE + REST POST Kommunikation

## Notizen fuer den Implementierer
- Die Datei heisst aktuell `LiveChartPage.tsx`. Umbenennung zu `LiveTradingView.tsx` ist sinnvoll fuer Konsistenz mit `BacktestLiveView.tsx`, aber optional. Route-Aenderungen beachten.
- `usePipelineStream` hat bereits das Event-Routing an den `tradingStore`. Es muss um den `onChartEvent` Callback und die `decisionEvents` Logik erweitert werden, nicht neu geschrieben.
- Im Live-Trading-Modus kommen `market-snapshot` Events ca. 1x pro Sekunde (Echtzeit). Das ist wesentlich langsamer als im Backtest — rAF-Batching ist dennoch korrekt, aber meistens werden einzelne Updates verarbeitet.
- Die Bar-Update-Logik (aktiven Bar aktualisieren vs. neuen Bar oeffnen) basiert auf der 1m-Zeitgrenze: Wenn der Timestamp des neuen Snapshots in denselben 1m-Slot faellt → updateActiveBar. Wenn neuer Slot → appendBar.
- ControlsPanel ist bereits implementiert und muss nur korrekt ins neue Layout integriert werden (links, feste Breite).

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.tsx`) fuer LiveTradingView Layout und Event-Routing
- [ ] Unit-Tests fuer tradingStore.addDecisionEvent und FIFO-Eviction
- [ ] Integrationstests fuer market-snapshot → Chart-Update Flow
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen geaenderten/neuen exportierten Funktionen
- [ ] CSS Modules mit Dark-Theme-Tokens
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
