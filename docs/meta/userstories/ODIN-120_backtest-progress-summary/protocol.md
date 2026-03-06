# ODIN-120 — BacktestProgressBar und BacktestDaySummary Komponenten

## Runde: R1 (Erstimplementierung)
**Datum:** 2026-03-06
**Commit:** `c367a34` auf `its-odin-ui/main`

---

## Deliverables

| # | Datei | Status |
|---|-------|--------|
| 1 | `src/domains/backtesting/components/BacktestProgressBar.tsx` | Erstellt |
| 2 | `src/domains/backtesting/components/BacktestProgressBar.module.css` | Erstellt |
| 3 | `src/domains/backtesting/components/BacktestDaySummary.tsx` | Erstellt |
| 4 | `src/domains/backtesting/components/BacktestDaySummary.module.css` | Erstellt |
| 5 | `src/domains/backtesting/utils/formatDuration.ts` | Erstellt |
| 6 | `src/domains/backtesting/utils/formatPnl.ts` | Erstellt |
| 7 | `src/domains/backtesting/components/BacktestProgressBar.test.tsx` | Erstellt |
| 8 | `src/domains/backtesting/components/BacktestDaySummary.test.tsx` | Erstellt |
| 9 | `src/domains/backtesting/utils/formatDuration.test.ts` | Erstellt |
| 10 | `src/domains/backtesting/utils/formatPnl.test.ts` | Erstellt |

## Build & Tests

- **tsc --noEmit:** CLEAN (0 Fehler in ODIN-120-Dateien)
- **npm run build:** GREEN (2.37s)
- **vitest run:** 54 Tests passed, 0 failed (4 Test-Dateien)

### Test-Aufteilung

| Datei | Tests |
|-------|-------|
| `formatDuration.test.ts` | 17 (inkl. NaN, Infinity, Boundary-Cases) |
| `formatPnl.test.ts` | 12 (inkl. Tausender-Trenner, negative Null) |
| `BacktestProgressBar.test.tsx` | 11 (Rendering, ARIA, Prozent-Clamping) |
| `BacktestDaySummary.test.tsx` | 14 (P&L-Formatierung, Farbcodierung, Drawdown) |

## Architektur-Entscheidungen

1. **Props-driven (keine Store-Kopplung):** Beide Komponenten empfangen alle Daten als Props. Die Verbindung zum `backtestLiveStore` erfolgt in der uebergeordneten Page-Komponente (BacktestLiveView, ODIN-121). Dies entspricht dem Konzept (12-live-chart-components.md, Story-Patterns).

2. **de-DE Locale fuer Zahlenformatierung:** `Intl.NumberFormat('de-DE')` fuer Tausender-Trenner und Dezimalkomma. Trader-optimiert fuer schnelle visuelle Erfassung grosser Werte.

3. **Bestehende Design Tokens:** CSS verwendet `--color-bg-overlay`, `--color-info`, `--color-positive`, `--color-negative` aus dem bestehenden Token-System (theme-dark.css). Die in der Story genannten `--progress-bar-bg` und `--pnl-positive` existieren nicht im Design System und wurden durch semantisch aequivalente Tokens ersetzt.

4. **Code-Sprache Englisch:** Labels ("Day", "Bar", "Calculating...") in Englisch gemaess CLAUDE.md-Regel "code/JavaDoc=English". Die deutschen Labels im Wireframe ("Tag") sind Dokumentationssprache, nicht Code.

## LLM-Sparring

### ChatGPT (1 Call)

**Wesentliche Findings und Massnahmen:**

| Finding | Massnahme |
|---------|-----------|
| `formatEstimatedDuration` Boundary-Bug: `hours` und `remainingMinutes` inkonsistent berechnet, kann `~1h 60 min` erzeugen | BEHOBEN: Ableitung von `hours` und `remainingMinutes` aus gerundeter `totalMinutes` statt aus Roh-Millisekunden |
| Drawdown bei 0 erhielt trotzdem `styles.negative` | BEHOBEN: Conditional Class basierend auf `maxDrawdown < 0` |
| Missing `aria-label` und `aria-valuetext` auf progressbar | BEHOBEN: `aria-label="Backtest progress"` und detaillierter `aria-valuetext` |
| `prefers-reduced-motion` fuer Progress-Transition | BEHOBEN: Media Query in CSS |
| NaN/Infinity-Handling in formatEstimatedDuration | BEHOBEN: `Number.isFinite()` Guard |

### Gemini D1 — Code Quality (Score: 4.5-5/5)

- Alle Dateien 4-5/5 bewertet
- Lob: readonly Modifiers, TSDoc, Constants, defensive Edge-Case-Handling
- Einziger Verbesserungsvorschlag: Hardcoded "EUR" Waehrung. AKZEPTIERT als zukuenftiger Parametrisierungs-Kandidat, derzeit keine Multi-Waehrungs-Anforderung.

### Gemini D2 — Konzepttreue

- Props-Interfaces: Exakte Uebereinstimmung mit Spec
- Visuelle Elemente: Alle Wireframe-Elemente vorhanden
- CSS Tokens: Deviation erkannt (Story-Tokens existieren nicht im Design System, durch bestehende ersetzt)
- Store-Integration: Als Deviation erkannt, aber konzept-konform (Props-driven ist korrekt laut Story-Patterns-Abschnitt)

### Gemini D3 — Praxis

- `tabular-nums` fuer Layout-Stabilitaet bei schnellen Updates positiv bewertet
- Transition-Risiko bei schnellen Updates: Bereits via `prefers-reduced-motion` adressiert
- Tausender-Trenner gefordert: UMGESETZT mit `Intl.NumberFormat('de-DE')`
- `overflow: hidden` fuer Container empfohlen: UMGESETZT
- Sparklines, Drawdown-%, Win/Loss-Split: Out of Scope (benoetigen zusaetzliche Props/Daten)

## Akzeptanzkriterien-Status

- [x] BacktestProgressBar zeigt visuellen Fortschrittsbalken mit Prozentsatz
- [x] Verarbeitete Tage / Gesamt Tage angezeigt
- [x] Verarbeitete Bars / Gesamt Bars angezeigt
- [x] Aktuelles Simulationsdatum angezeigt
- [x] Geschaetzte Restzeit formatiert (Minuten oder Stunden+Minuten)
- [x] BacktestDaySummary zeigt Tages-P&L, kumulativen P&L, Trades, Win-Rate, Max Drawdown
- [x] P&L-Werte farbkodiert (gruen positiv, rot negativ)
- [x] Props-driven (Store-Verbindung via ODIN-121)
- [x] CSS Modules mit Dark-Theme-Tokens
- [x] tsc --noEmit fehlerfrei
- [x] Unit-Tests fuer Formatierung und Rendering
