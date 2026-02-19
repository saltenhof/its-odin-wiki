# ODIN — Chart-Komponente: Architekturkonzept

Version: 1.0 DRAFT
Stand: 2026-02-19
Review: ChatGPT-Review R1 (10 Findings) + R2 (5 Findings) eingearbeitet

---

## 1. Zielsetzung und Einsatzszenarien

Die Chart-Komponente ist eine generische, wiederverwendbare Intraday-Chart-Ansicht fuer ODIN. Sie zeigt den Kursverlauf eines einzelnen Instruments fuer genau einen Handelstag, ueberlagert mit technischen KPIs und Trading-Events. Die Komponente wird in verschiedenen Domaenen eingesetzt und deckt zwei Betriebsmodi ab.

### Einsatzszenarien

| Szenario | Modus | Datenquelle | Typischer Nutzer |
|----------|-------|-------------|-----------------|
| **Live Trading** | Live | SSE-Stream (timeframe-konforme Bar-Updates) + REST (initiale Bars) | Operator waehrend des Handelstags |
| **Backtest-Analyse** | History | REST (`/api/v1/charts/{symbol}/bars`, `/indicators`, `/trades`) | Analyst nach Backtest-Lauf |
| **Vergangene Live-Tage** | History | REST (identische Endpoints, anderes `tradingDate`) | Operator bei Nachbetrachtung |
| **Sim-Replay** | Live (simuliert) | SSE-Stream (beschleunigte Events, `mode: SIMULATION`) | Analyst bei interaktiver Sim |

### Abgrenzung

- **Ein Instrument, ein Tag** — kein Multi-Day-Scrolling, kein Wochen-/Monats-View
- **Desktop-only** — kein Mobile-Support (Frontend-Guardrail §8)
- **Kein Analyse-Tool** — die Komponente visualisiert, sie berechnet keine KPIs selbst (Backend-Verantwortung)
- **Keine Zeichen-Tools** — keine Trendlinien, Fibonacci, Freihandzeichnung. Die Komponente zeigt vom Backend gelieferte Daten

---

## 2. Technologieentscheidung

### Empfehlung: TradingView Lightweight Charts v4

**Primaere Chart-Library:** [TradingView Lightweight Charts](https://tradingview.github.io/lightweight-charts/) (MIT-Lizenz, ~45 KB gzipped)

| Kriterium | Lightweight Charts | Recharts | D3.js | Chart.js |
|-----------|-------------------|----------|-------|----------|
| Candlestick-Support | Nativ, optimiert | Nicht nativ | Manuell | Plugin |
| Financial-Overlays | VWAP, EMA, BB nativ | Generisch | Manuell | Generisch |
| Performance (1000+ Bars) | WebGL-Rendering, exzellent | SVG, problematisch | SVG/Canvas, aufwaendig | Canvas, mittel |
| Inkrementelles Update | `update()` API nativ | Full Re-Render | Manuell | Re-Render |
| Zoom & Pan | Nativ, Mausrad + Drag | Manuell | Manuell | Plugin |
| Custom Primitives | Plugin-API fuer Custom Drawing | Nativ (generisch) | Volle Freiheit | Begrenzt |
| Bundle-Groesse | ~45 KB | ~120 KB | ~80 KB | ~60 KB |
| Trading-spezifisch | Ja (Kernzweck) | Nein | Nein | Nein |
| React-Wrapper | `lightweight-charts-react` | Nativ React | Kein offizieller | `react-chartjs-2` |

**Begruendung der Entscheidung:**

1. **Kernzweck ist Financial Charting.** Lightweight Charts wurde genau dafuer gebaut — Candlesticks, Volumen, Overlays, Time-Scale-Management. Andere Libraries erfordern erheblichen Custom-Code fuer das gleiche Ergebnis.
2. **Performance bei Echtzeit-Updates.** Die `update()`-API erlaubt inkrementelles Rendering einzelner Bars ohne Full-Re-Render — entscheidend fuer den Live-Modus mit 1-sekuendigen SSE-Updates.
3. **Bereits als Architekturentscheidung verankert.** Kap 0 §11 (A6) und Kap 9 §4.1 definieren TradingView Lightweight Charts als Chart-Library.
4. **Zoom und Pan nativ.** Mausrad-Zoom, Drag-Pan, Auto-Scaling — alles out-of-the-box, kein Custom-Code noetig.
5. **Plugin-API fuer Custom Primitives.** Lightweight Charts v4 unterstuetzt Custom Series Plugins und Custom Primitives — damit lassen sich Positions-Shading, vertikale Event-Linien und Stop-Level-Linien ohne Workarounds implementieren.

**Sekundaer-Library: Recharts** fuer nicht-finanzielle Charts (P&L-Verlauf, Quant-Score-History) — wie in Kap 9 §4.1 definiert. Nicht fuer den Haupt-Chart.

### React-Integration

Statt des offiziellen `lightweight-charts-react`-Wrappers wird ein **eigener duenner Wrapper-Hook** (`useChart`) entwickelt. Gruende:

- Der offizielle Wrapper ist ein Convenience-Layer, der die imperative API in deklarative React-Props uebersetzt. Fuer ODIN's Anforderungen (dynamische Layer, Live-Updates, Custom Primitives) brauchen wir direkten Zugriff auf die imperative API.
- Ein eigener Hook gibt volle Kontrolle ueber Lifecycle (Create, Update, Destroy), Memory-Management und Error-Handling.
- Kap 9 §4.1 und Frontend-Guardrail §7 fordern Hooks fuer Logik, nicht fuer Presentational Wrappers.

```typescript
// Skizze: useChart Hook
function useChart(
  containerRef: RefObject<HTMLDivElement>,
  options: ChartOptions
): ChartApi {
  // Erstellt IChartApi, managed Lifecycle, Cleanup bei Unmount
  // Gibt stabile Referenz zurueck fuer imperative Operationen
}
```

---

## 3. Chart-Architektur: Multi-Panel-Layout

### Panel-Modell

Inspiriert vom IB-Chart-Layout (siehe Referenzmaterial), aber vereinfacht fuer ODIN's Anforderungen. Der Chart besteht aus **vertikal gestapelten Panels**, wobei jedes Panel eine eigenstaendige Lightweight-Charts-Instanz ist.

```
┌─────────────────────────────────────────────────────────────────┐
│  Header: Instrument, OHLCV-Werte der aktuellen/letzten Kerze   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Panel 1: Preis-Chart (Candlesticks)                    [65%]   │
│  - Candlesticks (OHLCV)                                        │
│  - Preis-Overlays: VWAP, EMA(9), EMA(21), EMA(50), EMA(100)   │
│  - Bollinger Bands (Upper/Mid/Lower als zusammengehoeriger      │
│    Composite-Layer)                                             │
│  - Trading-Event-Markierungen (Entry, Exit, Partial Exit)       │
│  - Volumen als Balken am unteren Rand (im selben Panel)         │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Panel 2: Oszillatoren (RSI + ADX, zwei Y-Skalen)      [12%]   │
│  - RSI(14) mit Ueberverkauft/Ueberkauft-Zonen (30/70) [rechts] │
│  - ADX(14)                                              [rechts]│
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Panel 3: Volatilitaet                                  [12%]   │
│  - ATR(14)                                                      │
│  - ATR Decay                                                    │
│  - Volume Ratio                                                 │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Panel 4: Flow (optional)                               [11%]   │
│  - Accumulation/Distribution                     [linke Skala]  │
│  - OBV oder weitere Flow-Indikatoren (v2)        [linke Skala]  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Zeitachse (geteilt ueber alle Panels)                          │
│  [09:30]  [10:00]  [10:30]  [11:00]  ...  [15:30]  [16:00]    │
└─────────────────────────────────────────────────────────────────┘
```

> **Hinweis Panel 4 (Flow):** Dieses Panel ist optional und standardmaessig ausgeblendet. Es dient der Erweiterbarkeit fuer Volumen-/Flow-Indikatoren (Accumulation/Distribution, OBV), wie im IB-Referenz-Screenshot als mittleres Panel dargestellt. In v1 ist nur die Panel-Infrastruktur vorgesehen — die konkreten Flow-Indikatoren werden in v2 implementiert.

### Panel-Synchronisation

Alle Panels teilen sich dieselbe Zeitachse. Lightweight Charts unterstuetzt nativ **Time Scale Synchronisation** ueber die `timeScale()` API:

- **Scroll-Sync:** Horizontales Scrollen in einem Panel scrollt alle anderen Panels mit
- **Zoom-Sync:** Zoom in einem Panel zoomt alle anderen Panels identisch
- **Crosshair-Sync:** Die vertikale Crosshair-Linie ist ueber alle Panels synchronisiert

Die Synchronisation wird ueber einen dedizierten **`SyncController`** implementiert, der Reentrancy-Guards und robuste Null-Checks bietet:

```typescript
/**
 * Synchronizes time scale and crosshair across multiple chart panels.
 * Uses reentrancy guard to prevent feedback loops when one panel
 * triggers updates in others.
 */
class PanelSyncController {
  private readonly panels: IChartApi[] = [];
  private isSyncing: boolean = false;
  private readonly unsubscribers: Array<() => void> = [];

  addPanel(panel: IChartApi): void {
    this.panels.push(panel);
    this.setupSync(panel);
  }

  removePanel(panel: IChartApi): void {
    const index: number = this.panels.indexOf(panel);
    if (index >= 0) {
      this.panels.splice(index, 1);
      // Unsubscribe corresponding listeners
    }
  }

  private setupSync(sourcePanel: IChartApi): void {
    const unsubRange = sourcePanel.timeScale()
      .subscribeVisibleLogicalRangeChange((range: LogicalRange | null) => {
        if (this.isSyncing || range === null) return;
        this.isSyncing = true;
        try {
          for (const targetPanel of this.panels) {
            if (targetPanel === sourcePanel) continue;
            targetPanel.timeScale().setVisibleLogicalRange(range);
          }
        } finally {
          this.isSyncing = false;
        }
      });

    this.unsubscribers.push(unsubRange);
  }

  destroy(): void {
    for (const unsub of this.unsubscribers) {
      unsub();
    }
    this.unsubscribers.length = 0;
    this.panels.length = 0;
  }
}
```

### Multi-Scale Panels

Panels koennen **mehrere Y-Achsen** haben (wie im IB-Referenz-Screenshot: Accum/Dist auf der linken Skala, RSI/ADX auf der rechten Skala). Lightweight Charts unterstuetzt dies ueber `priceScaleId`:

- **Rechte Skala (Default):** Fuer die primaere Serie im Panel
- **Linke Skala:** Fuer Serien mit abweichendem Wertebereich (z.B. Accum/Dist neben RSI)

```typescript
// Beispiel: Zwei Skalen im Oszillator-Panel
rsiSeries.applyOptions({ priceScaleId: 'right' });
adxSeries.applyOptions({ priceScaleId: 'right' });  // Gleicher Wertebereich wie RSI
accumDistSeries.applyOptions({ priceScaleId: 'left' });  // Eigene Skala
```

### Panel-Konfigurierbarkeit

Panels 2, 3 und 4 sind **optional und toggle-bar**. Der Benutzer kann:

- Panels ein-/ausblenden (Checkbox in der Toolbar)
- Die vertikale Groesse per Drag-Resize aendern
- Die Panel-Aufteilung wird im LocalStorage persistiert

---

## 4. Layer-System

### Konzept: Layer als Composite-Abstraktion

Jeder Indikator ist ein **Layer**. Ein Layer ist nicht auf eine einzelne Chart-Serie beschraenkt — er kann mehrere Serien und/oder Custom Primitives umfassen. Dies ermoeglicht Composite-Layer (z.B. Bollinger Bands = 3 Linien) und nicht-serielle Overlays (z.B. Positions-Shading als Custom Primitive).

Layer werden ueber ein zentrales **Layer Registry** verwaltet. Neue KPIs koennen hinzugefuegt werden, indem ein neuer Layer registriert wird — ohne Umbau der Chart-Komponente.

```typescript
// Layer-Definition (Registry-Eintrag)
type ChartLayerConfig = {
  readonly id: LayerId;                    // Typ-sichere Layer-ID
  readonly label: string;                  // Anzeigename: 'EMA(9)', 'RSI(14)', 'VWAP'
  readonly panel: PanelType;               // Ziel-Panel
  readonly defaultVisible: boolean;        // Standardmaessig sichtbar?
  readonly create: (chart: IChartApi) => LayerInstance;  // Factory-Methode
};

// Layer-Instanz (zur Laufzeit)
type LayerInstance = {
  readonly series: readonly ISeriesApi<SeriesType>[];     // Eine oder mehrere Serien
  readonly primitives?: readonly ISeriesPrimitive[];      // Optionale Custom Primitives
  setData(initial: readonly LayerDataSlice[]): void;       // Bulk-Load (History oder initiales Laden)
  update(delta: LayerDataSlice): void;                     // Inkrementelles Update (Live-Modus)
  destroy(): void;                                         // Cleanup
};

// Panel-Typen
type PanelType = 'price' | 'oscillator' | 'volatility' | 'flow';

// Typ-sichere Layer-IDs (statt freier Strings)
type LayerId =
  | 'candlestick' | 'volume'
  | 'vwap' | 'ema9' | 'ema21' | 'ema50' | 'ema100'
  | 'bollinger'    // Composite: Upper + Mid + Lower
  | 'rsi14' | 'adx14'
  | 'atr14' | 'atr-decay' | 'volume-ratio'
  | 'accum-dist';
```

### Composite-Layer-Beispiel: Bollinger Bands

```typescript
function createBollingerLayer(chart: IChartApi): LayerInstance {
  const upperSeries: ISeriesApi<'Line'> = chart.addLineSeries({
    color: 'var(--color-kpi-bb)',
    lineStyle: LineStyle.Dashed,
    lineWidth: 1,
    priceScaleId: 'right',
  });
  const midSeries: ISeriesApi<'Line'> = chart.addLineSeries({
    color: 'var(--color-kpi-bb)',
    lineWidth: 1,
    priceScaleId: 'right',
  });
  const lowerSeries: ISeriesApi<'Line'> = chart.addLineSeries({
    color: 'var(--color-kpi-bb)',
    lineStyle: LineStyle.Dashed,
    lineWidth: 1,
    priceScaleId: 'right',
  });

  return {
    series: [upperSeries, midSeries, lowerSeries],
    setData(initial: readonly LayerDataSlice[]): void {
      // Filter null values during warmup — series will show gaps
      const upperData = initial
        .filter((d: LayerDataSlice) => d.bollingerUpper !== null)
        .map((d: LayerDataSlice) => ({ time: d.time, value: d.bollingerUpper! }));
      const midData = initial
        .filter((d: LayerDataSlice) => d.bollingerMid !== null)
        .map((d: LayerDataSlice) => ({ time: d.time, value: d.bollingerMid! }));
      const lowerData = initial
        .filter((d: LayerDataSlice) => d.bollingerLower !== null)
        .map((d: LayerDataSlice) => ({ time: d.time, value: d.bollingerLower! }));
      upperSeries.setData(upperData);
      midSeries.setData(midData);
      lowerSeries.setData(lowerData);
    },
    update(delta: LayerDataSlice): void {
      // Skip null values — do not push null into series (causes rendering errors)
      if (delta.bollingerUpper !== null) {
        upperSeries.update({ time: delta.time, value: delta.bollingerUpper });
      }
      if (delta.bollingerMid !== null) {
        midSeries.update({ time: delta.time, value: delta.bollingerMid });
      }
      if (delta.bollingerLower !== null) {
        lowerSeries.update({ time: delta.time, value: delta.bollingerLower });
      }
    },
    destroy(): void {
      chart.removeSeries(upperSeries);
      chart.removeSeries(midSeries);
      chart.removeSeries(lowerSeries);
    },
  };
}
```

### Non-Series Layer: Positions-Shading und Stop-Level

Overlays wie Positions-Shading und Stop-Level-Linien sind **keine traditionellen Zeitreihen-Serien**. Sie werden ueber Lightweight Charts' **Custom Primitives API** implementiert:

```typescript
// Position-Shading als Custom Primitive
function createPositionShadingLayer(chart: IChartApi, priceSeries: ISeriesApi<'Candlestick'>): LayerInstance {
  const shadingPrimitive: ISeriesPrimitive = new PositionShadingPrimitive();
  priceSeries.attachPrimitive(shadingPrimitive);

  return {
    series: [],
    primitives: [shadingPrimitive],
    update(data: LayerDataSlice): void {
      shadingPrimitive.updatePosition(data.positionEntry, data.positionExit, data.isProfitable);
    },
    destroy(): void {
      priceSeries.detachPrimitive(shadingPrimitive);
    },
  };
}
```

### Vordefinierte Layer (v1)

| Layer-ID | Panel | Komposition | Farbe | Default sichtbar |
|----------|-------|------------|-------|-----------------|
| `candlestick` | price | 1 Candlestick-Serie | Gruen/Rot | Immer |
| `volume` | price | 1 Histogram-Serie | Gruen/Rot (transparent) | Immer |
| `vwap` | price | 1 Line-Serie | `--color-kpi-vwap` (Cyan) | Ja |
| `ema9` | price | 1 Line-Serie | `--color-kpi-ema9` (Gelb) | Ja |
| `ema21` | price | 1 Line-Serie | `--color-kpi-ema21` (Orange) | Ja |
| `ema50` | price | 1 Line-Serie | `--color-kpi-ema50` (Blau) | Nein |
| `ema100` | price | 1 Line-Serie | `--color-kpi-ema100` (Violett) | Nein |
| `bollinger` | price | 3 Line-Serien (Upper/Mid/Lower) | `--color-kpi-bb` (Grau) | Nein |
| `rsi14` | oscillator | 1 Line-Serie + 2 PriceLines (30/70) | `--color-kpi-rsi` (Violett) | Ja |
| `adx14` | oscillator | 1 Line-Serie | `--color-kpi-adx` (Rot) | Ja |
| `atr14` | volatility | 1 Line-Serie | `--color-kpi-atr` (Tuerkis) | Ja |
| `atr-decay` | volatility | 1 Line-Serie | `--color-kpi-atr-decay` (Orange) | Nein |
| `volume-ratio` | volatility | 1 Line-Serie | `--color-kpi-vol-ratio` (Weiss) | Nein |
| `accum-dist` | flow | 1 Line-Serie (linke Skala) | `--color-kpi-accum-dist` (Gelb) | Nein |

### Layer Toggle UI

Eine **Toolbar** oberhalb des Charts zeigt alle verfuegbaren Layer mit Checkboxen. Jeder Layer-Eintrag zeigt:

- Farbiges Icon (Linie in der Layer-Farbe)
- Label
- Aktueller Wert (letzte Kerze)
- Checkbox fuer Sichtbarkeit

```
┌─────────────────────────────────────────────────────────┐
│ Toolbar: Layer Controls                                  │
│ [x] VWAP: 182.45  [x] EMA(9): 183.12  [x] EMA(21): .. │
│ [ ] EMA(50)  [ ] EMA(100)  [ ] BB  [x] RSI: 62.3       │
│ Timeframe: [1m] [5m] [15m]                               │
└─────────────────────────────────────────────────────────┘
```

### Erweiterbarkeit

Neue KPIs (z.B. AVWAP, EMA(200)) erfordern lediglich:

1. `LayerId`-Union um den neuen Identifier erweitern
2. Einen neuen Eintrag im Layer Registry (`ChartLayerConfig`) mit `create`-Factory
3. Das Backend liefert die Daten im `indicators`-Endpoint (REQ-CHART-002)
4. Keine Aenderung an der Chart-Rendering-Logik noetig

---

## 5. Trading-Event-Visualisierung

### Hybride Darstellung: Candle-Marker + vertikale Linien bei Fokus

Trading-Events werden als **Candle-Marker auf Preislevel** dargestellt — mit einer **vertikalen Meilenstein-Linie**, die erst bei Hover oder Selektion ueber alle Panels angezeigt wird. So wird der Preisbezug klar (Marker sitzt am Ausfuehrungspreis), ohne den Chart bei mehreren Events visuell zu ueberladen.

**Primaer-Darstellung (immer sichtbar):**

- **Entry:** Gruener Aufwaerts-Pfeil-Marker auf dem Candle am Ausfuehrungspreis
- **Full Exit:** Roter Abwaerts-Pfeil-Marker auf dem Candle am Ausfuehrungspreis
- **Partial Exit:** Kleinerer gelber Marker am Ausfuehrungspreis
- **Stop-Trigger:** Roter X-Marker am Stop-Preis

**Sekundaer-Darstellung (bei Hover/Selektion):**

- Vertikale gestrichelte Linie ueber alle Panels (Meilenstein-Stil)
- Annotation-Panel am oberen Rand des Preis-Charts mit Details

```
  │               ▲ @182.50              ▲ @185.20          ▼ @186.80
  │               │                      │                   │
  │     ░░░░░░░░░░│░░░░░░░░░░░░░░░░░░░░░│░░░░░░░░░░░░░░░░░░│░░░░░░
  │     Candles   BUY marker             SELL marker         EXIT marker
  │               (gruen, auf Preis)     (gelb, auf Preis)  (rot, auf Preis)
  │
  │  --- bei Hover auf einen Marker: ---
  │  ┌─────────────────────┐
  │  │ BUY 150 @182.50     │ ← Annotation-Panel
  │  │ Regime: TRENDING     │
  │  │ Score: 0.82          │
  │  └─────────────────────┘
  │               │ (vertikale Linie ueber alle Panels)
  ├───────────────┼──────────────────────────────────────────────────
  │  RSI Panel    │
  └───────────────┴──────────────────────────────────────────────────
```

### Event-Typen und Darstellung

| Event-Typ | Marker-Stil | Farbe | Hover-Annotation |
|-----------|-------------|-------|-----------------|
| Entry (Kauf) | Aufwaerts-Pfeil, gross | Gruen (`--color-event-entry`) | "BUY @{price}" + Stueckzahl + Regime + Score |
| Partial Exit | Abwaerts-Pfeil, klein | Gelb (`--color-event-partial`) | "SELL {qty} @{price} ({percent}%)" + Tranche |
| Full Exit | Abwaerts-Pfeil, gross | Rot (`--color-event-exit`) | "EXIT @{price}" + P&L + Grund |
| Stop-Trigger | X-Marker | Rot (`--color-event-stop`) | "STOP @{price}" + P&L |

### Detail-Popup bei Klick

Ein Klick auf einen Marker oeffnet ein **Detail-Popup** mit allen Informationen:

- tranchenVariant, exitReason, realizedPnl
- Regime und regimeConfidence zum Zeitpunkt des Events
- quantScore

### Positions-Hintergrund-Shading

Waehrend einer offenen Position wird der Hintergrund des Preis-Charts leicht eingefaerbt (via Custom Primitive):

- **Profitable Position:** Gruener Hintergrund (sehr dezent, ~5% Opazitaet)
- **Verlustposition:** Roter Hintergrund (sehr dezent, ~5% Opazitaet)
- **Bereich:** Von Entry-Zeitpunkt bis Exit-Zeitpunkt (oder bis zum aktuellen Zeitpunkt bei offener Position im Live-Modus)

### Stop-Level-Linie

Im Live-Modus und bei offener Position wird das aktuelle Stop-Level als **horizontale gestrichelte Linie** im Preis-Panel angezeigt (via `createPriceLine()`). Die Linie bewegt sich mit, wenn der Trailing-Stop nachgezogen wird.

### Datenquelle fuer Events

- **History-Modus:** `GET /api/v1/runs/{runId}/trades` (REQ-CHART-003)
- **Live-Modus:** `trade-update` und `order-update` SSE-Events (Kap 9 §3)

---

## 6. Datenmodell

### Chart-Daten-Interfaces

```typescript
// OHLCV Bar — Grundbaustein des Charts
type OhlcvBar = {
  readonly time: number;       // Unix-Timestamp (Sekunden) — Lightweight Charts Standard
  readonly open: number;
  readonly high: number;
  readonly low: number;
  readonly close: number;
  readonly volume: number;
};

// KPI-Snapshot pro Bar-Zeitpunkt
type IndicatorSnapshot = {
  readonly time: number;       // Unix-Timestamp, identisch mit Bar.time
  readonly ema9: number | null;
  readonly ema21: number | null;
  readonly ema50: number | null;
  readonly ema100: number | null;
  readonly rsi14: number | null;
  readonly atr14: number | null;
  readonly bollingerUpper: number | null;
  readonly bollingerMid: number | null;
  readonly bollingerLower: number | null;
  readonly adx14: number | null;
  readonly atrDecay: number | null;
  readonly volumeRatio: number | null;
  readonly vwap: number | null;
};

// Trading-Event fuer Chart-Marker
type ChartTradeEvent = {
  readonly time: number;       // Unix-Timestamp des Events
  readonly eventType: TradeEventType;
  readonly price: number;
  readonly quantity: number;
  readonly realizedPnl: number | null;    // Nur bei Exit
  readonly exitReason: string | null;     // Nur bei Exit
  readonly regime: string | null;
  readonly tranchenVariant: string | null;
};

type TradeEventType = 'ENTRY' | 'PARTIAL_EXIT' | 'FULL_EXIT' | 'STOP_TRIGGER';

// DecisionLog-Punkt fuer optionalen Decision-Layer
type ChartDecisionPoint = {
  readonly time: number;
  readonly intentType: string;       // ENTRY, EXIT, HOLD, etc.
  readonly finalDecision: string;    // APPROVED, REJECTED
  readonly rejectReason: string | null;
  readonly quantScore: number;
};

// Vollstaendiger Chart-Datensatz fuer einen Tag
type ChartDayData = {
  readonly instrumentId: string;
  readonly tradingDate: string;      // ISO date: '2026-01-15'
  readonly timeframe: Timeframe;
  readonly bars: readonly OhlcvBar[];
  readonly indicators: readonly IndicatorSnapshot[];
  readonly trades: readonly ChartTradeEvent[];
  readonly decisions: readonly ChartDecisionPoint[];  // Optional, on-demand
};

type Timeframe = '1m' | '5m' | '15m';
```

### Datenfluss

```
                    History-Modus                       Live-Modus
                    ─────────────                       ──────────

REST: /charts/{symbol}/bars ──┐                SSE: bar-update Event ──┐
REST: /charts/{symbol}/indicators ─┤            (timeframe-konforme    │
REST: /runs/{runId}/trades ───┤                  Bar-Updates)          │
REST: /runs/{runId}/decisions ┤                SSE: trade-update ──────┤
                               │                SSE: order-update ─────┤
                               v                                       v
                    ┌──────────────────┐              ┌──────────────────┐
                    │   ChartDataStore  │              │   ChartDataStore  │
                    │ (React State/Ref) │              │ (React State/Ref) │
                    │                  │              │                  │
                    │ bars: OhlcvBar[] │              │ bars: OhlcvBar[] │
                    │ indicators: ...  │              │ indicators: ...  │
                    │ trades: ...      │              │ trades: ...      │
                    └────────┬─────────┘              └────────┬─────────┘
                             │                                  │
                             v                                  v
                    ┌──────────────────────────────────────────────────┐
                    │            IntraDayChart Component                │
                    │                                                  │
                    │  useChart() → IChartApi                          │
                    │  Layer Registry → LayerInstance[]                 │
                    │  PanelSyncController → Time Scale Coordination   │
                    └──────────────────────────────────────────────────┘
```

---

## 7. Aggregationsvertrag: Backend liefert timeframe-konforme Bars

### Klare Verantwortungstrennung

> **Entscheidung:** Bar-Aggregation ist **ausschliesslich Backend-Verantwortung**. Das Frontend rendert ausschliesslich fertige, timeframe-konforme Bars. Es findet **keine Frontend-seitige Bar-Aggregation** statt.

Diese Entscheidung ist konsistent mit Kap 9 §4.1 ("Bar-Aggregation erfolgt backendseitig") und vermeidet die Komplexitaet von Frontend-Aggregationslogik, die schwer testbar und fehleranfaellig waere.

### Konsequenzen fuer REST und SSE

| Kanal | Verhalten |
|-------|----------|
| **REST** | `GET /api/v1/charts/{symbol}/bars?interval=FIVE_MINUTES` liefert fertige 5m-Bars |
| **SSE** | `snapshot`-Events enthalten `barOpenTime`, `barOpen`, `barHigh`, `barLow`, `lastPrice`, `barVolume` — bezogen auf den **subscribed Timeframe** |

### SSE-Subscription und Timeframe-Binding

Der SSE Instrument-Stream wird mit einem **Timeframe-Parameter** subscribed:

```
GET /api/v1/stream/instruments/{instrumentId}?timeframe=5m
```

Das Backend aggregiert die Roh-Ticks in den subscribed Timeframe und liefert timeframe-konforme Bar-Updates im `snapshot`-Event. Der Timeframe ist fuer die Lebensdauer der SSE-Verbindung fix.

**Bei Timeframe-Wechsel im Frontend (Overlap-Strategie, Details in §11):**

1. Neuen SSE-Stream mit neuem Timeframe-Parameter oeffnen (Buffer aktivieren)
2. Alten SSE-Stream schliessen — erst nachdem der neue Stream connected ist
3. Handover-Protokoll (§9) fuer den neuen Stream durchlaufen

> **Backend-Requirement:** Der SSE Instrument-Stream-Endpoint muss den Query-Parameter `timeframe` (Default: `5m`) unterstuetzen und die `snapshot`-Events entsprechend aggregiert liefern. Dies ist eine Erweiterung des bestehenden SSE-Contracts (Kap 9 §3).

### Kein Tick-zu-Bar-Mapping im Frontend

Das Frontend empfaengt niemals Roh-Ticks und baut niemals selbst Bars. Die `snapshot`-Events sind bereits Bar-bezogen:

- `barOpenTime` bestimmt, zu welcher Bar das Update gehoert
- `barOpen`, `barHigh`, `barLow`, `lastPrice` sind die aktuellen Werte der laufenden Bar
- Bei Bar-Wechsel (neuer `barOpenTime`) wird die vorherige Bar final und eine neue Bar beginnt

---

## 8. Zoom und Navigation

### Anforderung: Intraday-Zoom

Wenn erst 30 Minuten des Handelstages gelaufen sind (z.B. 6 Bars bei 5m), sollen diese 6 Bars **bildschirmfuellend** dargestellt werden — nicht als schmaler Streifen am linken Rand eines 6.5-Stunden-Charts.

### Loesung: Visible Range Management

Lightweight Charts bietet nativ zwei Zoom-Mechanismen:

**1. Auto-Fit (Default):**
Die Zeitachse passt sich automatisch an die vorhandenen Daten an. Wenn nur 30 Minuten Daten vorhanden sind, fuellen diese den gesamten sichtbaren Bereich. Dies ist das **Standardverhalten** und loest die Kernforderung direkt.

```typescript
// Auto-Fit: Alle vorhandenen Bars im sichtbaren Bereich
chart.timeScale().fitContent();
```

**2. Manueller Zoom (Mausrad + Drag):**
- **Mausrad:** Zoom in/out (horizontale Achse)
- **Drag:** Pan (horizontales Scrollen)
- **Doppelklick auf Zeitachse:** Reset auf Auto-Fit

**3. Zeitbereichs-Buttons (Quick-Zoom):**
Fuer schnellen Zugriff auf typische Intraday-Zeitfenster:

```
┌────────────────────────────────────────────┐
│  [30m] [1h] [2h] [Full Day] [Auto-Fit]    │
└────────────────────────────────────────────┘
```

Jeder Button setzt die sichtbare Zeitspanne. **Wichtig:** Der Algorithmus clampt den `from`-Wert auf die erste vorhandene Bar, damit bei wenig Daten kein leerer Raum links entsteht. Wenn die angeforderte Duration groesser ist als die verfuegbare Datenbreite, wird auf `fitContent()` zurueckgefallen:

```typescript
function setVisibleDuration(chart: IChartApi, durationMinutes: number, bars: readonly OhlcvBar[]): void {
  if (bars.length === 0) return;

  const firstBarTime: number = bars[0].time;
  const lastBarTime: number = bars[bars.length - 1].time;
  const requestedFrom: number = lastBarTime - (durationMinutes * 60);

  // Clamp: Nicht vor die erste Bar zoomen
  const effectiveFrom: number = Math.max(requestedFrom, firstBarTime);

  // Fallback: Wenn Duration > verfuegbare Daten, Auto-Fit
  if (effectiveFrom >= lastBarTime) {
    chart.timeScale().fitContent();
    return;
  }

  chart.timeScale().setVisibleRange({
    from: effectiveFrom as UTCTimestamp,
    to: lastBarTime as UTCTimestamp,
  });
}
```

### Live-Modus: Auto-Scroll

Im Live-Modus scrollt der Chart automatisch mit, sodass die neueste Kerze immer am rechten Rand sichtbar ist. Dieses Verhalten wird **deaktiviert**, sobald der Benutzer manuell scrollt oder zoomt (um historische Bars zu betrachten). Ein **"Live"-Button** am rechten Rand der Zeitachse springt zurueck zur aktuellen Kerze und reaktiviert Auto-Scroll.

```typescript
// Auto-Scroll Management
type AutoScrollState = {
  enabled: boolean;         // Derzeit aktiv?
  userOverridden: boolean;  // Hat der User manuell gescrollt?
};

function handleAutoScroll(chart: IChartApi, state: AutoScrollState): void {
  if (state.enabled && !state.userOverridden) {
    chart.timeScale().scrollToRealTime();
  }
}

// "Live"-Button: Reset auf Auto-Scroll
function resetToLive(chart: IChartApi, state: AutoScrollState): void {
  state.userOverridden = false;
  state.enabled = true;
  chart.timeScale().scrollToRealTime();
}
```

### Y-Achsen-Skalierung

- **Price-Panel:** Auto-Scale basierend auf sichtbaren Bars + Padding (5%)
- **Oszillator-Panel:** Feste Skala (RSI: 0-100, mit Zonen-Markierung bei 30/70). Sekundaere linke Skala fuer Flow-Indikatoren mit abweichendem Wertebereich
- **Volatilitaets-Panel:** Auto-Scale basierend auf sichtbaren Werten
- Manuelle Y-Achsen-Skalierung ist **nicht** vorgesehen (v1) — Auto-Scale deckt den Bedarf ab

---

## 9. Live-Update-Mechanik

### SSE-Integration

Im Live-Modus empfaengt der Chart Daten ueber den SSE Instrument-Stream (`/api/v1/stream/instruments/{instrumentId}`). Die relevanten Event-Typen:

| SSE-Event | Chart-Aktion |
|-----------|-------------|
| `snapshot` | Aktuelle Bar aktualisieren (OHLCV) oder neue Bar erstellen bei Bar-Wechsel |
| `trade-update` | Trading-Event-Marker hinzufuegen/aktualisieren |
| `order-update` | Stop-Level-Linie aktualisieren |

### Inkrementelles Bar-Update

SSE `snapshot`-Events enthalten timeframe-konforme Bar-Daten (§7). Der Chart unterscheidet zwei Faelle:

**Fall 1: Update innerhalb der aktuellen Bar**
Die letzte Kerze wird aktualisiert (High, Low, Close, Volume koennen sich aendern). Candlestick- und Volume-Serie werden parallel aktualisiert:

```typescript
function handleSnapshotUpdate(
  candlestickSeries: ISeriesApi<'Candlestick'>,
  volumeSeries: ISeriesApi<'Histogram'>,
  snapshot: MarketSnapshot
): void {
  candlestickSeries.update({
    time: snapshot.barOpenTime as UTCTimestamp,
    open: snapshot.barOpen,
    high: snapshot.barHigh,
    low: snapshot.barLow,
    close: snapshot.lastPrice,
  });

  volumeSeries.update({
    time: snapshot.barOpenTime as UTCTimestamp,
    value: snapshot.barVolume,
    color: snapshot.lastPrice >= snapshot.barOpen
      ? 'var(--color-candle-up)'
      : 'var(--color-candle-down)',
  });
}
```

**Fall 2: Neue Bar beginnt**
Wenn `snapshot.barOpenTime` groesser ist als die letzte bekannte Bar-Zeit, wird eine neue Kerze angelegt. **Wichtig:** `open` wird aus `snapshot.barOpen` gesetzt (nicht aus `lastPrice`), und die Volume-Serie wird parallel aktualisiert:

```typescript
function handleNewBar(
  candlestickSeries: ISeriesApi<'Candlestick'>,
  volumeSeries: ISeriesApi<'Histogram'>,
  snapshot: MarketSnapshot
): void {
  // Vorherige Bar ist jetzt final — kein weiteres Update
  // Neue Bar starten mit Backend-geliefertem barOpen
  candlestickSeries.update({
    time: snapshot.barOpenTime as UTCTimestamp,
    open: snapshot.barOpen,       // Backend-Open, nicht lastPrice
    high: snapshot.barHigh,
    low: snapshot.barLow,
    close: snapshot.lastPrice,
  });

  // Volume-Serie parallel aktualisieren (Reset auf neuen Bar-Startwert)
  volumeSeries.update({
    time: snapshot.barOpenTime as UTCTimestamp,
    value: snapshot.barVolume,
    color: snapshot.lastPrice >= snapshot.barOpen
      ? 'var(--color-candle-up)'
      : 'var(--color-candle-down)',
  });
}
```

### Indikator-Updates im Live-Modus

KPI-Werte werden pragmatisch in zwei Kanaelen geliefert:

- **Kompakte KPIs im `snapshot`-Event:** VWAP und RSI sind bereits im `snapshot`-Event enthalten (Kap 9 §3). Diese werden inkrementell aktualisiert.
- **Vollstaendige Indikatoren:** Periodisch per REST abgerufen (alle 60s oder bei Bar-Wechsel). Der REST-Endpoint liefert die neuesten Indikator-Werte. Nur die **sichtbaren Layer** werden per REST nachgeladen (Lazy-Loading).

### Initiales Laden und SSE-Handover-Protokoll

Der Uebergang von REST-Laden zu SSE-Updates ist ein kritischer Moment, an dem Datenverlust oder Doppelverarbeitung auftreten kann. Das folgende Protokoll stellt sicher, dass keine Events verloren gehen und keine Duplikate entstehen:

**Handover-Protokoll (4 Phasen):**

```
Phase 1: SSE-Stream oeffnen (SOFORT, vor REST-Calls)
  → SSE-Events werden in einen Client-seitigen Buffer geschrieben
  → Jedes Event hat eine eventId (monoton steigend, Kap 9 §3)

Phase 2: REST-Calls parallel ausfuehren
  → GET /charts/{symbol}/bars?tradingDate=today&interval=5m
  → GET /charts/{symbol}/indicators?tradingDate=today
  → GET /runs/{runId}/trades
  → REST-Responses enthalten ein `asOfEventId`-Feld (das hoechste eventId,
    das zum Zeitpunkt der REST-Abfrage bereits im SSE-Stream war)

Phase 3: REST-Daten rendern
  → Chart mit REST-Daten befuellen
  → asOfEventId merken

Phase 4: Buffer abarbeiten und in Live-Modus wechseln
  → Gepufferte SSE-Events mit eventId <= asOfEventId verwerfen (bereits in REST enthalten)
  → Gepufferte SSE-Events mit eventId > asOfEventId anwenden
  → Deduplizierung per barOpenTime: Wenn ein Bar-Update fuer eine bereits
    vorhandene barOpenTime kommt, wird nur das Update mit dem hoeheren eventId behalten
  → Ab jetzt: SSE-Events direkt verarbeiten (Buffer leer)
```

**Absicherungen:**

- **eventId-basierte Deduplizierung:** Jedes SSE-Event hat eine eindeutige eventId (monoton steigend pro Stream, Kap 9 §3). Events mit `eventId <= asOfEventId` werden verworfen.
- **barOpenTime als sekundaerer Key:** Fuer Bar-Updates wird `barOpenTime` als zusaetzlicher Deduplizierungsschluessel verwendet.
- **Timeout:** Wenn REST-Calls laenger als 10s dauern, wird eine Fehlermeldung angezeigt und der Chart zeigt nur SSE-Daten (ab dem Zeitpunkt des SSE-Verbindungsaufbaus).

### Reconnect-Semantik und eventId-Stabilitaet

Bei Verbindungsabbruechen waehrend des Handover-Protokolls oder im laufenden Betrieb muss die eventId-Monotonie erhalten bleiben. Das SSE-Protokoll (Kap 9 §3) definiert bereits:

- Jedes Event traegt eine `eventId` (monoton steigend, **pro Stream**)
- Frontend sendet `Last-Event-ID` beim Reconnect
- Backend replayed verpasste Events aus In-Memory-Ringbuffer (max. 60s)

**Zusaetzliche Garantien fuer den Chart-Handover:**

| Szenario | Verhalten |
|----------|----------|
| **Reconnect waehrend Phase 1-2 (Handover laeuft)** | Der gepufferte SSE-Stream wird verworfen. Handover startet komplett neu (Phase 1). |
| **Reconnect im laufenden Betrieb (Phase 4)** | Standard-SSE-Reconnect mit `Last-Event-ID`. Backend replayed verpasste Events. Der Chart aktualisiert nahtlos. |
| **Backend-Restart** | Ringbuffer leer, eventId-Sequenz startet bei 1 (Kap 9 §3). Frontend erkennt den Reset (eventId < letzte bekannte eventId). Reaktion: Full-Refresh — REST-Daten neu laden, gepufferte Events verwerfen, Handover neu starten. |
| **Timeframe-Wechsel** | Neuer SSE-Stream (§7, §11) = neue eventId-Sequenz. Der alte eventId-Zustand wird verworfen. Handover startet fuer den neuen Stream. |

> **Voraussetzung an das Backend:** Der REST-Response muss ein `asOfEventId`-Feld enthalten, das die Korrelation zwischen REST-Snapshot und SSE-Stream ermoeglicht. Die eventId-Sequenz muss innerhalb eines Handelstages und einer SSE-Verbindung monoton steigend sein. Dies muss als Backend-Requirement dokumentiert werden (Erweiterung von REQ-CHART-001).

---

## 10. Komponenten-API

### Props als Discriminated Union

Die Props verwenden **Discriminated Unions** (Frontend-Guardrail §4), damit modus-spezifische Pflichtfelder compiletime-sicher sind:

```typescript
// Gemeinsame Basis-Props
type ChartBaseProps = {
  readonly instrumentId: string;
  readonly tradingDate: string;             // ISO date
  readonly runId: string;                   // Pflicht in beiden Modi (fuer Trading-Events + KPI-Zuordnung)

  // Optionale Konfiguration
  readonly timeframe?: Timeframe;           // Default: '5m'
  readonly visibleLayers?: readonly LayerId[];  // Typ-sichere Layer-IDs
  readonly showTradingEvents?: boolean;     // Default: true
  readonly showDecisionPoints?: boolean;    // Default: false
  readonly initialZoom?: 'auto-fit' | 'last-hour' | 'last-30m';  // Default: 'auto-fit'

  // Panel-Konfiguration
  readonly showOscillatorPanel?: boolean;   // Default: true
  readonly showVolatilityPanel?: boolean;   // Default: true
  readonly showFlowPanel?: boolean;         // Default: false

  // Layout
  readonly className?: string;
  readonly panelHeights?: PanelHeightConfig;

  // Callbacks
  readonly onBarClick?: (bar: OhlcvBar) => void;
  readonly onTradeEventClick?: (event: ChartTradeEvent) => void;
  readonly onTimeframeChange?: (timeframe: Timeframe) => void;
  readonly onVisibleRangeChange?: (range: VisibleRange) => void;
  readonly onError?: (error: ChartError) => void;
};

// Discriminated Union: Live vs History
type IntraDayChartProps =
  | (ChartBaseProps & {
      readonly mode: 'live';
      readonly onConnectionStatusChange?: (status: ConnectionStatus) => void;
    })
  | (ChartBaseProps & {
      readonly mode: 'history';
    });

// Panel-Hoehen-Konfiguration
type PanelHeightConfig = {
  readonly price?: number;       // Prozent, Default: 65
  readonly oscillator?: number;  // Prozent, Default: 12
  readonly volatility?: number;  // Prozent, Default: 12
  readonly flow?: number;        // Prozent, Default: 11
};

type ConnectionStatus = 'connected' | 'reconnecting' | 'disconnected';

type VisibleRange = {
  readonly from: number;   // Unix-Timestamp
  readonly to: number;     // Unix-Timestamp
};
```

### Imperativer Zugriff (Ref)

Fuer Szenarien, in denen der Parent-Component den Chart programmatisch steuern muss (z.B. "Spring zu diesem Trade"):

```typescript
type IntraDayChartRef = {
  scrollToTime(time: number): void;
  setVisibleRange(from: number, to: number): void;
  fitContent(): void;
  toggleLayer(layerId: LayerId, visible: boolean): void;
  exportImage(): Promise<Blob>;     // Screenshot-Export
};
```

### Verwendung

```tsx
// History-Modus (Backtest-Analyse)
<IntraDayChart
  instrumentId="AAPL"
  tradingDate="2026-01-15"
  mode="history"
  runId="550e8400-e29b-41d4-a716-446655440000"
  timeframe="5m"
  showDecisionPoints={true}
/>

// Live-Modus (Operator-Dashboard)
<IntraDayChart
  instrumentId="AAPL"
  tradingDate="2026-02-19"
  mode="live"
  runId={currentRunId}
  timeframe="1m"
  onTradeEventClick={handleTradeDetail}
  onConnectionStatusChange={handleConnectionStatus}
/>
```

---

## 11. Timeframe-Wechsel

Der Chart unterstuetzt drei Timeframes: **1m**, **5m**, **15m** (Kap 9 §4.1, REQ-CHART-004).

### Verhalten bei Timeframe-Wechsel

Der Timeframe-Wechsel ist ein **zustandsbehafteter Uebergang**, der im Live-Modus Race Conditions vermeiden muss. Ein dedizierter `loadingState` steuert den Uebergang:

```typescript
type TimeframeSwitchState =
  | { status: 'IDLE'; activeTimeframe: Timeframe }
  | { status: 'SWITCHING'; targetTimeframe: Timeframe; previousTimeframe: Timeframe }
  | { status: 'ERROR'; failedTimeframe: Timeframe; activeTimeframe: Timeframe };
```

**Ablauf des Timeframe-Wechsels (Live-Modus):**

Der Timeframe-Wechsel folgt dem Handover-Protokoll (§9), mit einer **Overlap-Strategie** fuer die beiden SSE-Streams: Der neue Stream wird geoeffnet **bevor** der alte geschlossen wird. Damit gibt es kein Fenster, in dem `trade-update`- oder `order-update`-Events verloren gehen koennten.

1. **Status: SWITCHING** — Chart zeigt "Loading"-Indikator
2. **Neuen SSE-Stream oeffnen** mit neuem Timeframe-Parameter (§7) — Events werden sofort in einen Buffer geschrieben
3. **Alten SSE-Stream schliessen** (EventSource.close()) — erst nachdem der neue Stream connected ist (EventSource.readyState === OPEN). Dadurch entsteht ein kurzer Overlap, in dem beide Streams aktiv sind
4. **REST-Calls parallel ausfuehren** (Bars + Indicators im neuen Interval). REST-Responses enthalten `asOfEventId` fuer den **neuen** SSE-Stream
5. **Alle Series leeren und mit REST-Daten neu befuellen**
6. **Gepufferten Buffer abarbeiten** — nach Source-Tag und Event-Typ getrennt:
    - **`old-stream` Events:**
        - `snapshot`: Verwerfen (alter Timeframe, irrelevant)
        - `trade-update`: Via `tradeId`-Map sammeln (wird spaeter mit new-stream dedupliziert)
        - `order-update`: Via `orderId`-Map sammeln (wird spaeter mit new-stream dedupliziert)
    - **`new-stream` Events** — Handover-Protokoll (§9) anwenden:
        - Events mit `eventId <= asOfEventId`: Verwerfen (bereits in REST enthalten)
        - Events mit `eventId > asOfEventId`: Anwenden (snapshot, trade-update, order-update)
    - **Cross-Stream-Deduplizierung:** Fuer gesammelte `old-stream` trade/order Events: Nur anwenden, wenn kein `new-stream` Event mit gleicher `tradeId`/`orderId` existiert (Regel: new-stream gewinnt)
7. **Zoom-Position beibehalten** — der sichtbare Zeitbereich bleibt ungefaehr gleich (umgerechnet auf die neue Granularitaet)
8. **Status: IDLE** — SSE-Events werden direkt verarbeitet
9. **Bei Fehler: Status ERROR** — Neuen SSE-Stream schliessen, alten SSE-Stream (falls noch offen) weiter nutzen oder neuen mit altem Timeframe oeffnen. Fehlermeldung anzeigen.

> **Schluesselregeln:**
>
> - **Overlap-Strategie:** Der neue SSE-Stream wird **vor** dem Schliessen des alten geoeffnet. Erst wenn der neue Stream connected ist, wird der alte geschlossen. Damit gibt es kein Fenster ohne SSE-Verbindung — `trade-update` und `order-update` Events gehen nicht verloren.
> - **Stream-Tagging:** Jedes gepufferte Event wird mit einem Source-Tag (`old-stream` / `new-stream`) versehen. Beim Buffer-Drain:
>     - `old-stream` + `snapshot`: Verwerfen (alter Timeframe)
>     - `old-stream` + `trade-update` / `order-update`: Behalten und anwenden
>     - `new-stream`: Normales Handover-Protokoll (§9) — eventId-basierte Deduplizierung mit `asOfEventId`
> - **Neuer Stream vor REST:** Der neue SSE-Stream wird vor den REST-Calls geoeffnet, damit `asOfEventId` im REST-Response auf den neuen Stream korreliert ist.
> - **Cross-Stream-Deduplizierung (trade/order):** Waehrend des Overlaps koennen `trade-update` und `order-update` Events von **beiden** Streams eintreffen. Dedup-Regeln:
>     - **Primary Key:** `tradeId` fuer `trade-update`, `orderId` fuer `order-update`
>     - **Tie-Breaker:** `new-stream gewinnt` — bei gleicher fachlicher ID wird das Event vom neuen Stream bevorzugt und das vom alten Stream verworfen. Grund: Der neue Stream ist autoritativ ab dem Zeitpunkt der Subscription.
>     - **Implementation:** Ein einfaches `Map<string, Event>` (Key = tradeId/orderId) im Buffer. Beim Einfuegen: Wenn Key bereits vorhanden und Source = new-stream, ueberschreiben. Wenn Key bereits vorhanden und Source = old-stream, nur ueberschreiben wenn noch kein new-stream Event fuer diesen Key existiert.
>
> **Backend-Requirement:** `trade-update`-Events muessen eine stabile `tradeId` und `order-update`-Events eine stabile `orderId` tragen. Diese sind bereits implizit in Kap 9 §3 definiert (TradeUpdatePayload, OrderUpdatePayload), muessen aber explizit als Idempotenz-Keys fuer den Overlap-Mechanismus dokumentiert werden.

### Live-Modus und Timeframe

Im Live-Modus liefert das Backend die `snapshot`-Events bezogen auf den aktuell konfigurierten Timeframe (§7). Ein Timeframe-Wechsel erfordert:

- Neue REST-Daten fuer den gewaehlten Timeframe laden
- Die SSE-Events enthalten weiterhin `barOpenTime`-basierte Updates — das Frontend ordnet sie dem richtigen Timeframe zu

### History-Modus und Timeframe

Im History-Modus ist der Wechsel einfacher: Neuer REST-Call, Daten austauschen, fertig. Kein SSE-Handling noetig.

---

## 12. Session-Zeitfenster und Hintergrund-Shading

### Handelszeiten-Visualisierung

Aehnlich wie im IB-Referenz-Screenshot werden verschiedene Session-Phasen durch **Hintergrund-Shading** visualisiert (via Custom Primitive):

| Session-Phase | Zeitfenster (NYSE-Beispiel) | Hintergrund |
|---------------|---------------------------|-------------|
| Pre-Market | 04:00 - 09:30 ET | Dunkelbraun, 10% Opazitaet |
| Regular Trading Hours (RTH) | 09:30 - 16:00 ET | Kein Shading (Standard-Hintergrund) |
| After-Hours | 16:00 - 20:00 ET | Dunkelblau, 10% Opazitaet |

Die Session-Zeiten werden vom Backend als Konfiguration geliefert (abhaengig vom Exchange-Kalender, Kap 0 §3.2). Das Frontend rendert die Zonen als halbtransparente Rechtecke im Hintergrund aller Panels.

> **Hinweis:** In v1 handelt ODIN nur waehrend RTH. Pre-Market und After-Hours Daten werden dennoch angezeigt (falls vorhanden), um den Kontext zu geben — z.B. Gaps.

---

## 13. Integration in Domain-Kontexte

### Backtest-Feature (`src/features/history/`)

Die Chart-Komponente wird im History-Feature eingebettet, umgeben von zusaetzlichen Kontextinformationen:

```
┌──────────────────────────────────────────────────────────────┐
│ Run-Header: AAPL | 2026-01-15 | SIM | Batch: backtest-Q1   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              IntraDayChart (mode="history")             │  │
│  │              runId="..."  timeframe="5m"                │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ Summary: P&L: +$245.80 | Win Rate: 67% | Trades: 3          │
│ Decisions: [scrollbare DecisionLog-Tabelle]                  │
└──────────────────────────────────────────────────────────────┘
```

### Pipeline-Feature (`src/features/pipeline/`)

Im Live-Dashboard ist der Chart neben dem Pipeline-Status-Panel positioniert:

```
┌────────────────────────────────┬─────────────────────────────┐
│                                │  Pipeline Status             │
│  IntraDayChart                 │  State: POSITIONED          │
│  (mode="live")                 │  Price: 183.45 (+1.2%)      │
│                                │  Position: 150 @182.50      │
│                                │  P&L: +$142.50              │
│                                │  Stop: 181.20               │
│                                │  Regime: TRENDING (0.82)    │
│                                │                             │
│                                │  [Pause] [Resume]           │
└────────────────────────────────┴─────────────────────────────┘
```

### Komponentenort: `src/shared/components/IntraDayChart/`

Die Chart-Komponente ist ein **genuiner Shared-Baustein**, keine Feature-spezifische Logik. Sie gehoert nach `src/shared/components/IntraDayChart/` und ist damit von allen Features importierbar, ohne die Feature-Isolationsregel zu verletzen (Frontend-Guardrail §2).

> **Barrel-Export-Klarstellung:** Der Frontend-Guardrail §2 erlaubt Barrel-Exports auf Feature-Ebene. Fuer `shared/components/` wird diese Konvention analog angewandt: Ein einzelnes `index.ts` in `shared/components/IntraDayChart/` exportiert die Public API. Sub-Ordner (hooks/, layers/, panels/) haben **keine eigenen Barrel-Exports** — alle Imports innerhalb der Komponente sind relative Imports. Nur der Top-Level `index.ts` ist der oeffentliche Entry-Point.

---

## 14. Fehlerbehandlung

| Fehler | Verhalten |
|--------|----------|
| REST-Call fuer Bars schlaegt fehl | Placeholder mit Fehlermeldung im Chart-Bereich. Retry-Button. |
| SSE-Verbindung bricht ab | Reconnect mit Exponential Backoff (Frontend-Guardrail §6). Waehrend Reconnect: letzte bekannte Daten bleiben sichtbar, "Reconnecting..."-Banner ueber dem Chart. `onConnectionStatusChange`-Callback wird aufgerufen. |
| Keine Bars fuer den Tag vorhanden | Leerer Chart mit Meldung "Keine Daten fuer {instrument} am {date} verfuegbar". |
| Indikator-Daten nicht verfuegbar (null) | Layer wird mit Luecken dargestellt (Warmup-Phase bei RSI/EMA). Null-Werte werden uebersprungen, Linie setzt bei naechstem gueltigen Wert fort. |
| Backend liefert inkonsistente Timestamps | Frontend prueft monoton steigende Timestamps. Bei Verletzung: Warning im Dev-Console, Daten trotzdem rendern. |
| Timeframe-Wechsel schlaegt fehl | Rollback zum vorherigen Timeframe. Fehlermeldung in der Toolbar. |
| REST+SSE Handover: REST-Timeout | Nach 10s: Fehlermeldung, Chart zeigt nur SSE-Daten ab Verbindungszeitpunkt. |

---

## 15. Performance-Ueberlegungen

### Datenmenge pro Tag

| Timeframe | Bars (6.5h RTH) | Bars (mit Pre/After) | Datenpunkte (Bars + Indikatoren) |
|-----------|-----------------|---------------------|--------------------------------|
| 1m | ~390 | ~960 | ~13.000 (960 Bars x ~14 Felder) |
| 5m | ~78 | ~192 | ~2.700 |
| 15m | ~26 | ~64 | ~900 |

Diese Datenmengen sind fuer Lightweight Charts trivial — die Library ist fuer 10.000+ Bars ausgelegt.

### Optimierungen (nur bei nachgewiesenem Bedarf)

- **Debouncing von SSE-Updates:** Bei sehr hochfrequenten Snapshot-Events koennen Updates auf max. 1 pro 200ms debounced werden, um unnoetige Rendering-Calls zu vermeiden.
- **Lazy-Loading von Indikatoren:** Indikator-Daten per REST nur laden wenn der entsprechende Layer sichtbar ist.
- **Keine `React.memo` ohne Profiling:** Frontend-Guardrail §7 — erst messen, dann optimieren.

---

## 16. Farb-Schema und Design Tokens

Alle Chart-Farben werden ueber CSS Custom Properties definiert (Frontend-Guardrail §8). Dark Theme ist das einzige Theme.

```css
/* Chart-spezifische Design Tokens */
:root {
  /* Candlestick */
  --color-candle-up: #26a69a;        /* Gruen */
  --color-candle-down: #ef5350;      /* Rot */
  --color-candle-wick: #787b86;      /* Grau */

  /* KPI-Overlays */
  --color-kpi-vwap: #00bcd4;         /* Cyan */
  --color-kpi-ema9: #ffeb3b;         /* Gelb */
  --color-kpi-ema21: #ff9800;        /* Orange */
  --color-kpi-ema50: #2196f3;        /* Blau */
  --color-kpi-ema100: #9c27b0;       /* Violett */
  --color-kpi-bb: #9e9e9e;           /* Grau */
  --color-kpi-rsi: #ce93d8;          /* Helles Violett */
  --color-kpi-adx: #ff7043;          /* Rot-Orange */
  --color-kpi-atr: #4dd0e1;          /* Tuerkis */
  --color-kpi-atr-decay: #ffb74d;    /* Helles Orange */
  --color-kpi-vol-ratio: #e0e0e0;    /* Helles Grau */
  --color-kpi-accum-dist: #fdd835;   /* Gelb */

  /* Trading Events */
  --color-event-entry: #4caf50;       /* Gruen */
  --color-event-partial: #ffc107;     /* Gelb */
  --color-event-exit: #f44336;        /* Rot */
  --color-event-stop: #ff1744;        /* Intensives Rot */

  /* Session-Shading */
  --color-session-pre: rgba(121, 85, 72, 0.10);
  --color-session-after: rgba(33, 150, 243, 0.10);

  /* Position-Shading */
  --color-position-profit: rgba(76, 175, 80, 0.05);
  --color-position-loss: rgba(244, 67, 54, 0.05);

  /* Chart-Hintergrund */
  --color-chart-bg: #1e1e2e;
  --color-chart-grid: rgba(255, 255, 255, 0.06);
  --color-chart-text: #d4d4d4;
  --color-chart-crosshair: rgba(255, 255, 255, 0.3);
}
```

---

## 17. Dateistruktur

```
src/
├── shared/
│   └── components/
│       └── IntraDayChart/
│           ├── IntraDayChart.tsx           # Haupt-Komponente
│           ├── IntraDayChart.module.css    # Styling
│           ├── IntraDayChart.test.tsx      # Tests
│           ├── types.ts                    # OhlcvBar, IndicatorSnapshot, LayerId, etc.
│           ├── hooks/
│           │   ├── useChart.ts             # LW Charts Lifecycle-Hook
│           │   ├── useChartData.ts         # Daten-Laden (REST + SSE + Handover)
│           │   ├── usePanelSync.ts         # PanelSyncController-Hook
│           │   ├── useAutoScroll.ts        # Live-Modus Auto-Scroll
│           │   └── useTimeframeSwitch.ts   # Timeframe-Wechsel State Machine
│           ├── layers/
│           │   ├── layerRegistry.ts        # Layer-Definitionen und Registry
│           │   ├── LayerInstance.ts         # LayerInstance Interface
│           │   ├── priceOverlays.ts        # VWAP, EMA, BB Composite Layer-Setup
│           │   ├── oscillatorLayers.ts     # RSI, ADX Layer-Setup
│           │   ├── volatilityLayers.ts     # ATR, ATR Decay, Vol Ratio
│           │   └── flowLayers.ts           # Accum/Dist Layer-Setup
│           ├── primitives/
│           │   ├── PositionShadingPrimitive.ts  # Custom Primitive: Hintergrund-Shading
│           │   ├── SessionShadingPrimitive.ts   # Custom Primitive: Session-Zonen
│           │   └── StopLevelPrimitive.ts        # Custom Primitive: Stop-Level-Linie
│           ├── events/
│           │   ├── TradeEventMarkers.ts    # Candle-Marker + Hover-Linien
│           │   └── TradeEventPopup.tsx     # Detail-Popup bei Klick
│           ├── panels/
│           │   ├── PricePanel.tsx          # Panel 1: Candlesticks + Overlays
│           │   ├── OscillatorPanel.tsx     # Panel 2: RSI, ADX
│           │   ├── VolatilityPanel.tsx     # Panel 3: ATR, Vol Ratio
│           │   └── FlowPanel.tsx           # Panel 4: Accum/Dist (optional)
│           ├── sync/
│           │   └── PanelSyncController.ts  # Panel-Synchronisation mit Reentrancy-Guard
│           ├── toolbar/
│           │   ├── LayerToggle.tsx         # Layer ein-/ausblenden
│           │   ├── TimeframeSelector.tsx   # 1m / 5m / 15m
│           │   └── ZoomControls.tsx        # Quick-Zoom Buttons + Live-Button
│           └── index.ts                    # Barrel Export (einziger oeffentlicher Entry-Point)
```

> **Kein Barrel-Export in Sub-Ordnern.** Nur `index.ts` auf Komponenten-Ebene exportiert die Public API. Alle internen Imports sind relative Pfade.

---

## 18. Zusammenfassung der Architekturentscheidungen

| Entscheidung | Wahl | Begruendung |
|-------------|------|-------------|
| Chart-Library | TradingView Lightweight Charts v4 | Financial-Charting-Spezialist, nativ Candlestick + Zoom + Inkrementelle Updates + Custom Primitives. Kap 0 A6 |
| Sekundaer-Library | Recharts (P&L, Score) | Fuer nicht-finanzielle Charts. Kap 9 §4.1 |
| React-Integration | Eigener `useChart` Hook (kein offizieller Wrapper) | Volle Kontrolle ueber imperative API, Lifecycle, Memory |
| Panel-Modell | 4 Panels (Preis/Oszillator/Volatilitaet/Flow) mit synchronisierter Zeitachse | Inspiriert von IB-Referenz. Panel 4 (Flow) optional, fuer v2-Erweiterbarkeit |
| Panel-Synchronisation | `PanelSyncController` mit Reentrancy-Guard | Verhindert Feedback-Loops, robuste Null-Checks, sauberer Unmount |
| Multi-Scale | Linke + rechte Y-Achse pro Panel | Wie IB-Referenz: Accum/Dist links, RSI/ADX rechts |
| Layer-System | Composite-faehig: `LayerInstance` mit `series[]` + `primitives[]` | Erweiterbar: Bollinger = 3 Serien, Positions-Shading = Primitive. Typ-sichere `LayerId`-Union |
| Trading-Events | Hybrid: Candle-Marker (immer) + vertikale Linien (bei Hover) | Preisbezug durch Marker, visuelle Klarheit bei Hover, kein Clutter |
| Zoom-Mechanik | Auto-Fit + Mausrad + Quick-Zoom (mit Clamp auf erste Bar) | Auto-Fit loest das "30-Minuten-Problem". Clamp verhindert leere Zeitachse |
| Aggregationsvertrag | Backend liefert timeframe-konforme Bars, kein Frontend-Aggregation | Konsistent mit Kap 9 §4.1. Klare Verantwortungstrennung |
| REST+SSE Handover | eventId-basierte Deduplizierung + asOfEventId-Korrelation | Kein Datenverlust, keine Duplikate beim Uebergang |
| Timeframe-Wechsel | State Machine (IDLE/SWITCHING/ERROR) mit SSE-Buffering | Verhindert Race Conditions bei Live+Timeframe-Switch |
| Props-API | Discriminated Union (mode: 'live' / 'history'), typ-sichere LayerId | Compiletime-Sicherheit, Frontend-Guardrail §4 |
| Komponentenort | `src/shared/components/IntraDayChart/` | Genuiner Shared-Baustein, keine Feature-spezifische Logik |
| Barrel-Export | Nur auf Komponenten-Top-Level, keine Sub-Ordner-Barrels | Konsistent mit Frontend-Guardrail §2 |
| Farben | CSS Custom Properties (Design Tokens) | Konsistent mit Dark Theme, einfach aenderbar |

---

## Offene Punkte / Backend-Requirements

Die folgenden Punkte erfordern Backend-Aenderungen oder -Erweiterungen:

| Punkt | Beschreibung | Ziel-WP |
|-------|-------------|---------|
| `asOfEventId` in REST-Responses | Bars- und Indicators-Endpoints muessen ein `asOfEventId`-Feld zurueckgeben, das den korrespondierenden SSE-Event-Stand angibt. Voraussetzung fuer das Handover-Protokoll (§9). | WP-04, WP-06 |
| SSE Timeframe-Parameter | Der SSE Instrument-Stream-Endpoint muss den Query-Parameter `timeframe` (Default: `5m`) unterstuetzen und `snapshot`-Events mit timeframe-konformen Bar-Daten liefern. Erweiterung des bestehenden SSE-Contracts (Kap 9 §3). | WP-08 |
| eventId-Stabilitaet | eventIds muessen innerhalb eines Handelstages und einer SSE-Verbindung monoton steigend sein. Bei Backend-Restart startet die Sequenz bei 1 (bereits definiert in Kap 9 §3). | WP-08 |
| Session-Zeiten-Endpoint | `GET /api/v1/exchanges/{exchangeId}/session` mit Pre-Market, RTH, After-Hours Zeiten. | Neu |
| Flow-Indikatoren (v2) | Accumulation/Distribution als Indikator in der KPI-Engine und im Indicators-Endpoint. | WP-06 (v2) |

---

## Referenzen

- [Kap 0 — Systemuebersicht](../../architecture/00-system-overview.md): Ports, Modi, MarketClock
- [Kap 9 — React-Frontend](09-frontend.md): SSE-Events, REST-Endpoints, Chart-Ansicht-Definition
- [Frontend-Guardrail](../guardrails/frontend.md): Projektstruktur, TypeScript-Regeln, Styling
- [Web Application Requirements](../requirements/web-application-requirements.md): REQ-CHART-001 bis REQ-CHART-005
- [TradingView Lightweight Charts Dokumentation](https://tradingview.github.io/lightweight-charts/)
