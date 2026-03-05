## Kontext

Weder im Live-Trading noch im Backtest existiert eine chronologische Auflistung der Pipeline-Entscheidungen. Der Operator sieht Marker im Chart, aber nicht das Reasoning dahinter. Das DecisionFeed ist ein neues Shared-Component, das LLM-Analysen, Quant-Scores, Gate-Results, Order-Events und Alerts als filterbaren, virtualisierten Activity-Log darstellt. Es wird sowohl in der BacktestLiveView als auch in der LiveTradingView eingesetzt.

## Modul

its-odin-ui — `src/shared/components/DecisionFeed/`

## Abhaengigkeiten

Keine (reine Darstellungskomponente, empfaengt Events als Props)

## Scope

### In Scope
- `DecisionFeed` Container-Komponente mit `react-window` Virtualisierung (FixedSizeList)
- `DecisionFeedItem` mit farbcodierten Kategorien (LLM=blau, Quant=gruen, Gate=gelb, Order=cyan, Alert=rot) und aufklappbaren JSON-Details (lazy-formatiert, memoized)
- `DecisionFeedFilter` mit Single-Select Tabs (LLM/Quant/Gate/Order/Alert/Alle)
- Auto-scroll Freeze: wenn User scrollt → Pause, "N new events" Indicator-Badge
- Chart-Feed Linking: `onEventClick` Callback, der den Event-Timestamp zurueckgibt (Chart kann Crosshair positionieren)
- Memory-Limit: max 1000 Events (FIFO Eviction), konfiguriert via `MEMORY_LIMITS.DECISION_EVENTS_MAX`

### Out of Scope
- SSE-Subscription (Domain-Hook-Aufgabe)
- Backtest-spezifische oder Trading-spezifische Integration
- Store-Management (Events werden als Props uebergeben)

## Akzeptanzkriterien
- [ ] `DecisionFeed` rendert eine virtualisierte Liste von Events mit `react-window`
- [ ] `DecisionFeedItem` zeigt Timestamp, Kategorie-Badge (farbcodiert), Summary-Text und aufklappbare Details
- [ ] JSON-Details werden lazy formatiert (erst bei Aufklappen) und per `React.memo` gecacht
- [ ] `DecisionFeedFilter` zeigt Tabs: Alle, LLM, Quant, Gate, Order, Alert — Single-Select
- [ ] Filter-Wechsel filtert die angezeigte Liste korrekt
- [ ] Auto-scroll ist aktiv: neue Events scrollen automatisch rein
- [ ] Auto-scroll pausiert bei manuellem User-Scroll, "N new events" Badge erscheint
- [ ] Klick auf Badge scrollt zurueck nach unten und reaktiviert Auto-scroll
- [ ] `onEventClick` Callback wird mit Event-Timestamp aufgerufen
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer Filter-Logik, Auto-scroll-State und Event-Click-Callback existieren

## Technische Details

### Neue Dateien
- `src/shared/components/DecisionFeed/DecisionFeed.tsx`:
  ```typescript
  interface DecisionFeedProps {
    readonly events: ReadonlyArray<DecisionFeedEvent>;
    readonly activeFilter: DecisionCategory | 'all';
    readonly onFilterChange: (filter: DecisionCategory | 'all') => void;
    readonly onEventClick?: (timestamp: string) => void;
    readonly className?: string;
  }
  ```
  Container mit `FixedSizeList` aus `react-window`. Filtert Events nach `activeFilter`. Verwaltet Auto-scroll-State.

- `src/shared/components/DecisionFeed/DecisionFeedItem.tsx`:
  ```typescript
  interface DecisionFeedItemProps {
    readonly event: DecisionFeedEvent;
    readonly onClick?: (timestamp: string) => void;
    readonly style: React.CSSProperties; // von react-window
  }
  ```
  Einzelner Feed-Eintrag. Aufklappbar via Chevron-Icon. JSON-Details werden mit `JSON.stringify(detail, null, 2)` formatiert — aber erst bei Aufklappen (lazy), und per `React.memo` + `useMemo` gecacht.

- `src/shared/components/DecisionFeed/DecisionFeedFilter.tsx`:
  ```typescript
  interface DecisionFeedFilterProps {
    readonly activeFilter: DecisionCategory | 'all';
    readonly onFilterChange: (filter: DecisionCategory | 'all') => void;
    readonly eventCounts: Readonly<Record<DecisionCategory | 'all', number>>;
  }
  ```
  Tab-Bar mit Counts pro Kategorie.

- `src/shared/components/DecisionFeed/types.ts`:
  ```typescript
  type DecisionCategory = 'llm' | 'quant' | 'gate' | 'order' | 'alert';

  interface DecisionFeedEvent {
    readonly id: string;
    readonly category: DecisionCategory;
    readonly timestamp: string;
    readonly instrumentId: string;
    readonly summary: string;
    readonly detail: Record<string, unknown>;
  }
  ```

- `src/shared/components/DecisionFeed/DecisionFeed.module.css` — CSS Module mit Dark-Theme-Tokens
- `src/shared/components/DecisionFeed/DecisionFeedItem.module.css`
- `src/shared/components/DecisionFeed/DecisionFeedFilter.module.css`
- `src/shared/components/DecisionFeed/index.ts` — Barrel Export

### Farbcodierung (CSS Custom Properties)
```css
--decision-color-llm: #4a9eff;      /* Blau */
--decision-color-quant: #4caf50;     /* Gruen */
--decision-color-gate: #ff9800;      /* Orange/Gelb */
--decision-color-order: #00bcd4;     /* Cyan */
--decision-color-alert: #f44336;     /* Rot */
```

### Patterns
- **Virtualisierung:** `react-window` FixedSizeList mit Item-Hoehe ~48px (collapsed) / ~200px (expanded). Expandierte Items nutzen `VariableSizeList` oder feste Hoehe mit Scroll innerhalb des Items.
- **Auto-scroll:** `listRef.current.scrollToItem(events.length - 1)` nach jedem neuen Event, AUSSER `isUserScrolled === true`. Scroll-Event-Listener erkennt User-Scroll via `onScroll` Callback.
- **Lazy JSON:** `JSON.stringify` wird in `useMemo` mit `event.detail` als Dependency gecacht. Formatierung erfolgt erst wenn `isExpanded === true`.
- **FIFO Eviction:** Die aufrufende Komponente ist fuer das Limit verantwortlich (max 1000 Events). DecisionFeed selbst rendert was es bekommt.

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §6: DecisionFeed-Architektur, §6.2: Virtualisierung mit react-window, §6.3: Filter-Design (Single-Select Tabs), §6.5: Chart-Feed Linking, §3.4: Props-Interface DecisionFeedProps

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §2: Shared-Components in shared/components/, §4: TypeScript strict, readonly, §7: Komponenten-Regeln (Props-Interface, keine Business-Logik), §8: CSS Modules, Dark Theme
- `CLAUDE.md` — Frontend Architecture: CSS Modules, dark theme, desktop-only

## Notizen fuer den Implementierer
- `react-window` muss als Dependency installiert werden (`npm install react-window @types/react-window`). Pruefen ob bereits vorhanden.
- Die Farbcodierung der Kategorien sollte als CSS Custom Properties definiert werden, nicht als Inline-Styles. So kann das Theme-System sie ueberschreiben.
- Der `onEventClick` Callback ist optional — er wird nur verwendet wenn der DecisionFeed neben einem Chart eingesetzt wird. Der Chart positioniert dann seinen Crosshair zum Event-Timestamp.
- Die Virtualisierung ist wichtig, weil bei Backtests tausende Events generiert werden. Ohne Virtualisierung wuerde die DOM-Node-Anzahl explodieren.
- `FixedSizeList` ist einfacher als `VariableSizeList`. Fuer expandierte Items: entweder feste Max-Hoehe mit innerem Scroll, oder Wechsel zu `VariableSizeList` mit dynamischer Hoehenmessung.
- Memory-Limit (1000 Events) wird vom aufrufenden Code durchgesetzt, nicht vom DecisionFeed selbst. Das DecisionFeed ist eine reine Darstellungskomponente.

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.tsx`) fuer Filter-Logik, Auto-scroll-State, Event-Click
- [ ] Integrationstests fuer DecisionFeed mit gemockten Events (Rendering, Filterung, Scroll)
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen exportierten Komponenten und Interfaces
- [ ] CSS Modules mit Dark-Theme-Tokens
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
