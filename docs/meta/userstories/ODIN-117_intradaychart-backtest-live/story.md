## Kontext

Der IntraDayChart bekommt einen dritten Modus `backtest-live` fuer Live-Streaming waehrend Backtests. Aktuell unterstuetzt der Chart nur `live` und `history`. Im neuen Modus streamt der Chart Bars in Echtzeit waehrend eines laufenden Backtests, mit Day-Rollover-Reset und Auto-Scroll.

## Modul

its-odin-ui — `src/shared/components/IntraDayChart/`

## Abhaengigkeiten

#9 (ODIN-116: useChartLiveUpdates)

## Scope

### In Scope
- `ChartMode` Union erweitern: `'live' | 'history' | 'backtest-live'`
- IntraDayChart Props anpassen: neuer Modus in der Mode-Prop, optionaler `onChartEvent` Callback
- `useChartLiveUpdates` Integration im `backtest-live` Modus
- Day-Rollover-Reset: Bei Tageswechsel Chart zuruecksetzen, Rolling-Window mit letzten 30 Bars des Vortags grau dargestellt
- Auto-Scroll (Follow-Modus): Chart scrollt automatisch mit neuen Bars mit, Freeze-Option wenn User manuell scrollt
- Exchange-Timezone Konfiguration in der Chart `timeScale` (z.B. US/Eastern fuer US-Equities)

### Out of Scope
- SSE-Subscription (Domain-Hook-Aufgabe, ODIN-119)
- DecisionFeed (ODIN-118)
- Reconnect-Recovery (ODIN-123)

## Akzeptanzkriterien
- [ ] `ChartMode` Type akzeptiert `'backtest-live'` als dritten Wert
- [ ] IntraDayChart rendert korrekt im `backtest-live` Modus
- [ ] `useChartLiveUpdates` wird im `backtest-live` Modus aktiviert
- [ ] Day-Rollover setzt den Chart zurueck: `setData()` mit neuem Tag, letzte 30 Bars des Vortags grau als Kontext
- [ ] Auto-Scroll ist aktiv: Chart folgt dem letzten Bar
- [ ] Auto-Scroll pausiert wenn User manuell scrollt/zoomt, mit visuellem "Follow"-Button zum Reaktivieren
- [ ] Exchange-Timezone ist in der timeScale konfiguriert
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer Day-Rollover-Logik und Auto-Scroll-State existieren

## Technische Details

### Geaenderte Dateien
- `src/shared/components/IntraDayChart/types.ts` (oder wo ChartMode definiert ist):
  ```typescript
  type ChartMode = 'live' | 'history' | 'backtest-live';
  ```

- `src/shared/components/IntraDayChart/IntraDayChart.tsx`:
  - Props erweitern um optionalen `onChartEvent` Callback
  - Conditional Logic fuer `backtest-live`: useChartLiveUpdates aktivieren, Auto-Scroll starten
  - Day-Rollover Handler: bei neuem `currentDate` aus Progress-Event → `setData()` mit grauem Vortags-Kontext

- `src/shared/components/IntraDayChart/hooks/useChartData.ts`:
  - Im `backtest-live` Modus: initialer REST-Load des aktuellen simulierten Tages, dann Live-Updates via Callbacks

### Neue Dateien
- `src/shared/components/IntraDayChart/hooks/useAutoScroll.ts`:
  ```typescript
  export function useAutoScroll(
    chartRef: React.RefObject<IChartApi>,
    enabled: boolean,
  ): {
    readonly isFollowing: boolean;
    readonly resumeFollow: () => void;
  };
  ```
  Implementiert Follow-Modus: `chart.timeScale().scrollToRealTime()` nach jedem Update. Registriert `subscribeVisibleLogicalRangeChange` um User-Scroll zu erkennen und Follow zu pausieren.

- `src/shared/components/IntraDayChart/utils/dayRollover.ts`:
  ```typescript
  export function prepareDayRolloverData(
    previousDayBars: ReadonlyArray<OhlcvBar>,
    contextBarCount: number,
  ): ReadonlyArray<OhlcvBar>;
  ```
  Nimmt die letzten N Bars des Vortags und markiert sie als Kontext (grau-Darstellung via Series-Customization).

### Patterns
- **Day-Rollover:** Wenn `currentDate` sich aendert, wird `series.setData()` aufgerufen (nicht `update()`). Die letzten 30 Bars des Vortags werden als graue Kontext-Bars vorangestellt. `chartEpoch` wird inkrementiert.
- **Auto-Scroll:** `chart.timeScale().scrollToRealTime()` wird nach jedem `enqueueUpdate` aufgerufen, AUSSER der User hat manuell gescrollt. Ein "Follow"-Button am Chart-Rand reaktiviert den Follow-Modus.
- **Timezone:** `timeScale: { timeVisible: true, secondsVisible: false, timezone: exchangeTimezone }` in der Chart-Konfiguration.

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §5: Chart-Update-Strategie, §10: Dual-Use (Live + Backtest), §10.4: Day-Change UX (Rolling-Window), §11.2: Timezone-Konfiguration
- `docs/concept/12-live-chart-components.md` — §2.2: Backtest-Live-View Wireframe (mode=backtest-live)

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §4: TypeScript strict, Union Types, §7: Komponenten-Regeln (Props-Interface, keine Business-Logik in Komponenten)
- `CLAUDE.md` — Frontend Architecture: Union Types statt enum, CSS Modules, dark theme

## Notizen fuer den Implementierer
- Die `backtest-live` und `live` Modi teilen sich den `useChartLiveUpdates` Hook — der Unterschied liegt nur in der Datenquelle (Domain-Hook) und im Day-Rollover-Verhalten.
- Die grauen Kontext-Bars des Vortags sind ein visueller Hinweis, kein funktionales Feature. Sie koennen via `color`-Override in der Candlestick-Series realisiert werden.
- `scrollToRealTime()` ist eine Methode der Lightweight Charts API. Sie scrollt zur rechten Kante des Charts. Achtung: Wenn der User gerade in einem anderen Bereich schaut, ist erzwungenes Scrollen stoerend — daher der Freeze-Mechanismus.
- Die Exchange-Timezone muss als Prop an IntraDayChart uebergeben werden (Default: 'America/New_York' fuer US-Equities).
- Bei der Day-Rollover-Implementierung: alle Indikator-Serien muessen ebenfalls zurueckgesetzt werden, nicht nur die Candlestick-Series.

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.ts`) fuer Day-Rollover-Logik, Auto-Scroll-State, ChartMode-Handling
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
