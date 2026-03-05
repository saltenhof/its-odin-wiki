## Kontext

Fortschrittsanzeige fuer laufende Backtests. Waehrend eines Backtest-Laufs (30-90 Minuten) benoetigt der Operator visuelles Feedback: Wie weit ist der Backtest? Wie performt die Strategie pro Tag? Die `BacktestProgressBar` zeigt Prozent, verarbeitete/gesamt Tage und Bars, sowie geschaetzte Restzeit. Die `BacktestDaySummary` zeigt die kompakte P&L-Zusammenfassung des letzten abgeschlossenen Handelstages.

## Modul

its-odin-ui — `src/domains/backtesting/components/`

## Abhaengigkeiten

#12 (ODIN-119: backtestLiveStore)

## Scope

### In Scope
- `BacktestProgressBar` Komponente: Prozent-Balken, verarbeitete/gesamt Tage, verarbeitete/gesamt Bars, aktuelles Simulationsdatum, geschaetzte Restzeit (formatiert als "~42 min" oder "~1h 23 min")
- `BacktestDaySummary` Komponente: Kompakte Tages-P&L-Zusammenfassung (Tages-P&L, kumulativer P&L, Anzahl Trades, Win-Rate, Max Drawdown)
- Integration mit `backtest-bar-progress` und `backtest-day-summary` Events via backtestLiveStore Selectors
- CSS Modules mit Dark-Theme-Tokens
- Farbkodierung: positiver P&L gruen, negativer P&L rot

### Out of Scope
- SSE-Subscription (kommt via backtestLiveStore / useBacktestStream)
- Chart (ODIN-117)
- DecisionFeed (ODIN-118)
- BacktestLiveView Page-Layout (ODIN-121)

## Akzeptanzkriterien
- [ ] `BacktestProgressBar` zeigt einen visuellen Fortschrittsbalken mit korrektem Prozentsatz
- [ ] Verarbeitete Tage / Gesamt Tage werden angezeigt (z.B. "Tag 47/120")
- [ ] Verarbeitete Bars / Gesamt Bars werden angezeigt (z.B. "Bar 195/390")
- [ ] Aktuelles Simulationsdatum wird angezeigt
- [ ] Geschaetzte Restzeit wird formatiert angezeigt (Minuten oder Stunden+Minuten)
- [ ] `BacktestDaySummary` zeigt Tages-P&L, kumulativen P&L, Trades, Win-Rate und Max Drawdown
- [ ] P&L-Werte sind farbkodiert (gruen positiv, rot negativ)
- [ ] Beide Komponenten lesen Daten aus `backtestLiveStore` via Zustand Selectors
- [ ] CSS Modules mit Dark-Theme-Tokens verwendet
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer Formatierung (Restzeit, P&L, Prozent) und Rendering existieren

## Technische Details

### Neue Dateien
- `src/domains/backtesting/components/BacktestProgressBar.tsx`:
  ```typescript
  interface BacktestProgressBarProps {
    readonly completedDays: number;
    readonly totalDays: number;
    readonly completedBars: number;
    readonly totalBars: number;
    readonly currentDate: string;
    readonly estimatedRemainingMs: number | null;
  }
  ```
  Reine Darstellungskomponente. Fortschrittsbalken als `<div>` mit prozentualer Breite. Restzeit-Formatierung als Utility-Funktion.

- `src/domains/backtesting/components/BacktestProgressBar.module.css`

- `src/domains/backtesting/components/BacktestDaySummary.tsx`:
  ```typescript
  interface BacktestDaySummaryProps {
    readonly tradingDate: string;
    readonly dayPnl: number;
    readonly cumulativePnl: number;
    readonly totalTrades: number;
    readonly winRate: number;
    readonly maxDrawdown: number;
  }
  ```
  Kompakte Inline-Darstellung: alle Werte in einer Zeile mit Labels.

- `src/domains/backtesting/components/BacktestDaySummary.module.css`

- `src/domains/backtesting/utils/formatDuration.ts`:
  ```typescript
  /** Formats milliseconds to human-readable duration ("~42 min", "~1h 23 min"). */
  export function formatEstimatedDuration(ms: number | null): string;
  ```

- `src/domains/backtesting/utils/formatPnl.ts`:
  ```typescript
  /** Formats P&L value with sign and currency symbol. */
  export function formatPnl(value: number): string;
  ```

### CSS Custom Properties
```css
--progress-bar-bg: var(--color-surface-secondary);
--progress-bar-fill: var(--color-accent-primary);
--pnl-positive: #4caf50;
--pnl-negative: #f44336;
```

### Patterns
- **Props-driven:** Beide Komponenten sind reine Darstellungskomponenten. Sie empfangen alle Daten als Props. Die Verbindung zum Store erfolgt in der uebergeordneten Page-Komponente (BacktestLiveView).
- **Formatierung:** Separate Utility-Funktionen fuer Dauer und P&L-Formatierung — testbar und wiederverwendbar.

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §7: Fortschrittsanzeige (ProgressBar + DaySummary), §3.4: Props-Interfaces BacktestProgressBarProps und BacktestDaySummaryProps, §2.2: Backtest-Live-View Wireframe (Proportionen ~80px)

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §7: Komponenten-Regeln (Props-Interface, keine Business-Logik), §8: CSS Modules, Dark Theme, §3: Naming-Konventionen
- `CLAUDE.md` — Frontend Architecture: CSS Modules, dark theme, desktop-only

## Notizen fuer den Implementierer
- Die `estimatedRemainingMs` kann `null` sein (z.B. am Anfang des Backtests, wenn noch keine Schaetzung moeglich ist). In dem Fall "Berechne..." oder aehnliches anzeigen.
- Win-Rate wird als Dezimalzahl (0.0 - 1.0) uebergeben und muss als Prozent formatiert werden (z.B. "62.5%").
- Max Drawdown ist ein negativer Wert (z.B. -235.40). Anzeige als absoluter Wert mit "DD" Label.
- Die Komponenten sind bewusst schlank gehalten. Keine komplexe Logik — nur Formatierung und Darstellung.
- Beide Komponenten werden in ODIN-121 (BacktestLiveView) oberhalb des Charts platziert, zusammen ~80px Hoehe.

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.tsx`) fuer Rendering beider Komponenten mit verschiedenen Props
- [ ] Unit-Tests (Vitest: `*.test.ts`) fuer `formatEstimatedDuration` und `formatPnl` Utility-Funktionen
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen exportierten Komponenten und Funktionen
- [ ] CSS Modules mit Dark-Theme-Tokens
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
