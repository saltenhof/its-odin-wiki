# ODIN — Frontend-Architekturkonzept: Design System & Domain-Driven Modulstruktur

**Version:** 1.0
**Stand:** 2026-02-19
**Review:** 3 Runden ChatGPT-Architektur-Review (R1: Grundstruktur, R2: Token-Governance/Event-Bus/SSE-Contract, R3: Finalisierung)

---

## 1. Zielsetzung und Leitprinzipien

### 1.1 Zielsetzung

Dieses Dokument definiert das Architekturkonzept fuer die ODIN React-Frontend-Anwendung mit zwei Schwerpunkten:

1. **Zentrales Design System** — einheitliche visuelle Sprache, zentral steuerbar, unabhaengig von einzelnen Features
2. **Domain-Driven Frontend-Architektur** — fachliche Domaenen als architektonische Einheiten, erweiterbar ohne Strukturbruch

Das Konzept ist ein Implementierungsleitfaden: konkret genug fuer den Entwicklungsbeginn, generisch genug fuer zukuenftige Erweiterungen.

### 1.2 Leitprinzipien

| Prinzip | Bedeutung |
|---------|-----------|
| **Domain-First** | Die Ordnerstruktur bildet fachliche Domaenen ab, nicht technische Schichten. Jede Domaene ist ein geschlossenes Modul |
| **Design System als Infrastruktur** | Farben, Typografie, Spacing, Theming sind zentrale Infrastruktur — kein Feature entscheidet ueber visuelle Grundlagen |
| **Kein Cross-Domain-Import** | Analog zum Backend (Fachmodul->Fachmodul verboten). Domaenen kommunizieren ausschliesslich ueber den zentralen Event-Bus |
| **Desktop-Only** | Kein Responsive fuer Mobile. Layouts optimiert fuer >= 1920x1080. Schriftgroessen beziehen sich auf die Monitor-/OS-Aufloesung (px-basiert), nicht auf die Browserbreite |
| **SSE-First State** | Der SSE-Stream IST der primaere State. REST fuer On-Demand-Daten. Kein redundanter Client-Cache |
| **Lean** | Keine Abstraktionsschichten ohne Auftrag. Erst bauen, dann messen, dann optimieren |

### 1.3 Abgrenzung

- **Kap 9** definiert die Backend-API-Contracts (SSE-Events, REST-Endpoints) — dieses Dokument beschreibt die Frontend-Architektur
- **Frontend-Guardrail** definiert die Bauvorschriften (TypeScript-Regeln, Naming, Tests) — dieses Dokument definiert die Struktur und das Design System
- **Keine UI-Wireframes** — dieses Dokument spezifiziert Architektur, nicht Visual Design

### 1.4 Normative Quellen und Konfliktaufloesung

Dieses Dokument baut auf bestehenden normativen Dokumenten auf und ueberschreibt in spezifischen Punkten bisherige Regelungen:

| Aenderung | Bisher (Guardrail/Kap 9) | Neu (dieses Dokument) | Begruendung |
|-----------|-------------------------|----------------------|-------------|
| **State Management** | "Kein Redux / Zustand" (Guardrail §5) | Zustand fuer Store-basiertes State Management | Erweiterter Scope (Backtesting, Cross-Domain) erfordert Selector-basiertes Re-Render; React Context allein fuehrt zu Performance-Problemen bei SSE-Updates |
| **Projektstruktur** | Feature-basiert: `src/features/*` (Guardrail §2) | Domain-basiert: `src/domains/*` mit Features als Substruktur | Uebergeordnete fachliche Domaenen ermoeglichen saubere Boundaries und Erweiterbarkeit |
| **Theme-Policy** | "Dark Theme — Einziges Theme" (Guardrail §8) | Dark als primaeres Theme (Default), Light Theme als optionale Erweiterung | Dark bleibt Default und Pflicht. Light ist vorbereitet (Token-Architektur stuetzt es), aber keine Implementierungspflicht in v1 |
| **API-Pfade** | `/api/stream/...` (Kap 9) | `/api/v1/stream/...` | Angleichung an API-Versionierung (REQ-WEB-005 aus Requirements-Dokument) |

Der **Frontend-Guardrail** muss nach Annahme dieses Konzepts in diesen vier Punkten aktualisiert werden. Bis zur Aktualisierung gilt dieses Dokument als autoritativ fuer die genannten Abweichungen.

---

## 2. Design System Architektur

### 2.1 Ueberblick

Das Design System ist eine zentrale Schicht, die alle visuellen Grundlagen definiert. Features konsumieren diese Grundlagen ueber CSS Custom Properties (Design Tokens) und UI Components. Kein Feature definiert eigene Grundfarben, Schriftgroessen oder Spacing-Werte.

```
┌─────────────────────────────────────────────────────┐
│                  Domain / Feature Code                │
│         (konsumiert Tokens + UI Components)           │
├─────────────────────────────────────────────────────┤
│                   UI Components                       │
│   (shared/components — konsumieren Semantic Tokens)   │
├─────────────────────────────────────────────────────┤
│               Semantic Design Tokens                  │
│      (theme-abhaengig, einzige Referenz fuer Code)    │
├─────────────────────────────────────────────────────┤
│               Primitive Design Tokens                 │
│         (Rohwerte, nie direkt referenziert)            │
├─────────────────────────────────────────────────────┤
│                    Theme Provider                      │
│          (data-theme auf html, localStorage)          │
└─────────────────────────────────────────────────────┘
```

**Ownership-Klaerung:** Die UI Components Library (`shared/components/`) ist Teil des Design Systems im konzeptionellen Sinne, lebt aber in `shared/components/` (nicht in `design-system/`), weil sie TypeScript-Code enthalten und der `design-system/`-Ordner reine CSS-Dateien fuehrt.

### 2.2 Token-Architektur (Drei Ebenen)

Design Tokens folgen einer dreistufigen Hierarchie: Primitive -> Semantic -> Component.

#### Ebene 1: Primitive Tokens (Farbpalette, Typografie-Skala, Spacing-Skala)

Primitive Tokens definieren die Rohwerte — unabhaengig von ihrer Verwendung. Sie werden **nie direkt in Feature-Code oder CSS Modules referenziert**. Enforcement: Stylelint-Rule verbietet `var(--p-*)` ausserhalb von `tokens/`.

```css
/* tokens/primitives.css */

:root {
  /* --- Farben (Neutral-Palette) --- */
  --p-gray-50:  #fafafa;
  --p-gray-100: #f5f5f5;
  --p-gray-200: #e5e5e5;
  --p-gray-300: #d4d4d4;
  --p-gray-400: #a3a3a3;
  --p-gray-500: #737373;
  --p-gray-600: #525252;
  --p-gray-700: #404040;
  --p-gray-800: #262626;
  --p-gray-850: #1e1e1e;
  --p-gray-900: #171717;
  --p-gray-950: #0a0a0a;

  /* --- Farben (Signal-Palette) --- */
  --p-green-400: #4ade80;
  --p-green-500: #22c55e;
  --p-green-600: #16a34a;
  --p-red-400:   #f87171;
  --p-red-500:   #ef4444;
  --p-red-600:   #dc2626;
  --p-blue-400:  #60a5fa;
  --p-blue-500:  #3b82f6;
  --p-blue-600:  #2563eb;
  --p-yellow-400: #facc15;
  --p-yellow-500: #eab308;
  --p-orange-400: #fb923c;
  --p-orange-500: #f97316;
  --p-purple-400: #c084fc;
  --p-purple-500: #a855f7;
  --p-cyan-400:  #22d3ee;
  --p-teal-400:  #2dd4bf;
  --p-pink-400:  #f472b6;
  --p-white:     #ffffff;
  --p-black:     #000000;

  /* --- Typografie-Skala (px-basiert, Desktop-only) --- */
  --p-font-size-xs:   11px;
  --p-font-size-sm:   12px;
  --p-font-size-base: 14px;
  --p-font-size-md:   16px;
  --p-font-size-lg:   18px;
  --p-font-size-xl:   20px;
  --p-font-size-2xl:  24px;
  --p-font-size-3xl:  30px;

  /* --- Schriftfamilien --- */
  --p-font-family-sans:  'Inter', 'Segoe UI', system-ui, -apple-system, sans-serif;
  --p-font-family-mono:  'JetBrains Mono', 'Cascadia Code', 'Fira Code', monospace;

  /* --- Schriftgewichte --- */
  --p-font-weight-regular: 400;
  --p-font-weight-medium:  500;
  --p-font-weight-semibold: 600;
  --p-font-weight-bold:    700;

  /* --- Line Heights --- */
  --p-line-height-tight:  1.25;
  --p-line-height-normal: 1.5;
  --p-line-height-relaxed: 1.625;

  /* --- Spacing-Skala (4px Base Grid) --- */
  --p-space-0:   0;
  --p-space-1:   4px;
  --p-space-2:   8px;
  --p-space-3:   12px;
  --p-space-4:   16px;
  --p-space-5:   20px;
  --p-space-6:   24px;
  --p-space-8:   32px;
  --p-space-10:  40px;
  --p-space-12:  48px;
  --p-space-16:  64px;

  /* --- Border Radius --- */
  --p-radius-none: 0;
  --p-radius-sm:   2px;
  --p-radius-md:   4px;
  --p-radius-lg:   8px;
  --p-radius-xl:   12px;
  --p-radius-full: 9999px;

  /* --- Border Widths --- */
  --p-border-width-none: 0;
  --p-border-width-thin: 1px;
  --p-border-width-medium: 2px;
  --p-border-width-thick: 3px;

  /* --- Opacity --- */
  --p-opacity-disabled: 0.4;
  --p-opacity-overlay:  0.6;
  --p-opacity-hover:    0.08;

  /* --- Schatten --- */
  --p-shadow-sm:  0 1px 2px rgba(0, 0, 0, 0.3);
  --p-shadow-md:  0 4px 6px rgba(0, 0, 0, 0.3);
  --p-shadow-lg:  0 10px 15px rgba(0, 0, 0, 0.3);

  /* --- Transitions --- */
  --p-transition-fast:   100ms ease;
  --p-transition-normal: 200ms ease;
  --p-transition-slow:   300ms ease;

  /* --- Z-Index-Skala --- */
  --p-z-base:      0;
  --p-z-dropdown:  100;
  --p-z-sticky:    200;
  --p-z-overlay:   300;
  --p-z-modal:     400;
  --p-z-toast:     500;
}
```

#### Ebene 2: Semantic Tokens (Theme-abhaengig)

Semantic Tokens weisen den Primitives eine **Bedeutung** zu. Sie wechseln je nach Theme (Dark/Light). Feature-Code referenziert ausschliesslich Semantic Tokens.

```css
/* tokens/theme-dark.css */

[data-theme='dark'] {
  /* --- Hintergruende --- */
  --color-bg-app:        var(--p-gray-950);
  --color-bg-surface:    var(--p-gray-900);
  --color-bg-elevated:   var(--p-gray-850);
  --color-bg-overlay:    var(--p-gray-800);
  --color-bg-hover:      var(--p-gray-800);
  --color-bg-active:     var(--p-gray-700);
  --color-bg-selected:   var(--p-gray-800);
  --color-bg-disabled:   var(--p-gray-900);

  /* --- Text --- */
  --color-text-primary:    var(--p-gray-100);
  --color-text-secondary:  var(--p-gray-400);
  --color-text-tertiary:   var(--p-gray-500);
  --color-text-disabled:   var(--p-gray-600);
  --color-text-inverse:    var(--p-gray-900);
  --color-text-link:       var(--p-blue-400);

  /* --- Borders --- */
  --color-border-default:  var(--p-gray-800);
  --color-border-strong:   var(--p-gray-600);
  --color-border-focus:    var(--p-blue-500);

  /* --- Focus Ring --- */
  --focus-ring-color:      var(--p-blue-500);
  --focus-ring-width:      var(--p-border-width-medium);
  --focus-ring-offset:     2px;

  /* --- Borders --- */
  --border-width-default:  var(--p-border-width-thin);
  --border-width-strong:   var(--p-border-width-medium);

  /* --- Opacity --- */
  --opacity-disabled:      var(--p-opacity-disabled);
  --opacity-overlay:       var(--p-opacity-overlay);

  /* --- Signalfarben (Trading-spezifisch) --- */
  --color-positive:        var(--p-green-400);
  --color-positive-hover:  var(--p-green-500);
  --color-negative:        var(--p-red-400);
  --color-negative-hover:  var(--p-red-500);
  --color-warning:         var(--p-yellow-400);
  --color-info:            var(--p-blue-400);
  --color-accent:          var(--p-orange-400);
  --color-accent-hover:    var(--p-orange-500);

  /* --- Pipeline-State-Farben (verbindlich, Kap 9 §4.2) --- */
  --color-state-startup:      var(--p-gray-500);
  --color-state-warmup:       var(--p-yellow-400);
  --color-state-observing:    var(--p-blue-400);
  --color-state-positioned:   var(--p-green-400);
  --color-state-pending-fill: var(--p-orange-400);
  --color-state-halted:       var(--p-red-400);
  --color-state-day-stopped:  var(--p-gray-600);

  /* --- Alert-Level-Farben --- */
  --color-alert-info:      var(--p-gray-500);
  --color-alert-warning:   var(--p-yellow-400);
  --color-alert-critical:  var(--p-red-400);

  /* --- Chart-spezifische Farben (vollstaendige Liste, abgestimmt mit Chart-Komponenten-Konzept §16) --- */
  --color-chart-candle-up:       var(--p-green-400);
  --color-chart-candle-down:     var(--p-red-400);
  --color-chart-volume:          var(--p-gray-600);
  --color-chart-grid:            var(--p-gray-800);
  --color-chart-crosshair:       var(--p-gray-500);

  /* KPI-Overlays (Preis-Panel) */
  --color-chart-ema9:            var(--p-blue-400);
  --color-chart-ema21:           var(--p-purple-400);
  --color-chart-ema50:           var(--p-cyan-400);
  --color-chart-ema100:          var(--p-teal-400);
  --color-chart-vwap:            var(--p-orange-400);
  --color-chart-bollinger:       var(--p-gray-500);

  /* KPI-Overlays (Oszillator-/Volatilitaets-/Flow-Panel) */
  --color-chart-rsi:             var(--p-purple-400);
  --color-chart-adx:             var(--p-red-400);
  --color-chart-atr:             var(--p-cyan-400);
  --color-chart-atr-decay:       var(--p-orange-400);
  --color-chart-vol-ratio:       var(--p-gray-400);
  --color-chart-accum-dist:      var(--p-yellow-400);

  /* Trading-Event-Marker */
  --color-chart-entry-marker:    var(--p-green-400);
  --color-chart-partial-marker:  var(--p-yellow-400);
  --color-chart-exit-marker:     var(--p-red-400);
  --color-chart-stop-level:      var(--p-red-400);
  --color-chart-decision-dot:    var(--p-yellow-400);

  /* Session-Shading (Pre-Market / After-Hours Hintergrund) */
  --color-chart-session-pre:     rgba(121, 85, 72, 0.10);
  --color-chart-session-after:   rgba(33, 150, 243, 0.10);

  /* Position-Shading (offene Position Hintergrund) */
  --color-chart-position-profit: rgba(74, 222, 128, 0.05);
  --color-chart-position-loss:   rgba(248, 113, 113, 0.05);

  /* --- Chart-Serienfarben (fuer Multi-Series-Vergleiche, z.B. Equity Curves) --- */
  --color-chart-series-1: var(--p-blue-400);
  --color-chart-series-2: var(--p-green-400);
  --color-chart-series-3: var(--p-orange-400);
  --color-chart-series-4: var(--p-purple-400);
  --color-chart-series-5: var(--p-cyan-400);
  --color-chart-series-6: var(--p-pink-400);

  /* --- Typografie (Semantic) --- */
  --font-family-body:    var(--p-font-family-sans);
  --font-family-code:    var(--p-font-family-mono);
  --font-size-body:      var(--p-font-size-base);
  --font-size-label:     var(--p-font-size-sm);
  --font-size-caption:   var(--p-font-size-xs);
  --font-size-heading-1: var(--p-font-size-3xl);
  --font-size-heading-2: var(--p-font-size-2xl);
  --font-size-heading-3: var(--p-font-size-xl);
  --font-size-heading-4: var(--p-font-size-lg);
  --font-weight-body:    var(--p-font-weight-regular);
  --font-weight-label:   var(--p-font-weight-medium);
  --font-weight-heading: var(--p-font-weight-semibold);
  --line-height-body:    var(--p-line-height-normal);
  --line-height-heading: var(--p-line-height-tight);

  /* --- Layout --- */
  --layout-sidebar-width:   240px;
  --layout-header-height:   48px;
  --layout-content-padding: var(--p-space-6);
  --layout-panel-gap:       var(--p-space-4);
}
```

```css
/* tokens/theme-light.css (optional, nicht Pflicht fuer v1) */

[data-theme='light'] {
  --color-bg-app:        var(--p-gray-50);
  --color-bg-surface:    var(--p-white);
  --color-bg-elevated:   var(--p-gray-100);
  --color-bg-overlay:    var(--p-gray-200);
  --color-bg-hover:      var(--p-gray-100);
  --color-bg-active:     var(--p-gray-200);
  --color-bg-selected:   var(--p-blue-400);
  --color-bg-disabled:   var(--p-gray-100);

  --color-text-primary:    var(--p-gray-900);
  --color-text-secondary:  var(--p-gray-600);
  --color-text-tertiary:   var(--p-gray-500);
  --color-text-disabled:   var(--p-gray-400);
  --color-text-inverse:    var(--p-white);
  --color-text-link:       var(--p-blue-600);

  --color-border-default:  var(--p-gray-200);
  --color-border-strong:   var(--p-gray-400);
  --color-border-focus:    var(--p-blue-500);

  --focus-ring-color:      var(--p-blue-500);
  --focus-ring-width:      var(--p-border-width-medium);
  --focus-ring-offset:     2px;

  --border-width-default:  var(--p-border-width-thin);
  --border-width-strong:   var(--p-border-width-medium);

  --opacity-disabled:      var(--p-opacity-disabled);
  --opacity-overlay:       var(--p-opacity-overlay);

  --color-positive:        var(--p-green-600);
  --color-positive-hover:  var(--p-green-500);
  --color-negative:        var(--p-red-600);
  --color-negative-hover:  var(--p-red-500);
  --color-warning:         var(--p-yellow-500);
  --color-info:            var(--p-blue-600);
  --color-accent:          var(--p-orange-500);
  --color-accent-hover:    var(--p-orange-400);

  --color-state-startup:      var(--p-gray-500);
  --color-state-warmup:       var(--p-yellow-500);
  --color-state-observing:    var(--p-blue-500);
  --color-state-positioned:   var(--p-green-500);
  --color-state-pending-fill: var(--p-orange-500);
  --color-state-halted:       var(--p-red-500);
  --color-state-day-stopped:  var(--p-gray-400);

  --color-alert-info:      var(--p-gray-500);
  --color-alert-warning:   var(--p-yellow-500);
  --color-alert-critical:  var(--p-red-500);

  /* --- Chart-spezifische Farben (abgestimmt mit Chart-Komponenten-Konzept §16) --- */
  --color-chart-candle-up:       var(--p-green-500);
  --color-chart-candle-down:     var(--p-red-500);
  --color-chart-volume:          var(--p-gray-300);
  --color-chart-grid:            var(--p-gray-200);
  --color-chart-crosshair:       var(--p-gray-400);

  --color-chart-ema9:            var(--p-blue-500);
  --color-chart-ema21:           var(--p-purple-500);
  --color-chart-ema50:           var(--p-cyan-400);
  --color-chart-ema100:          var(--p-teal-400);
  --color-chart-vwap:            var(--p-orange-500);
  --color-chart-bollinger:       var(--p-gray-400);

  --color-chart-rsi:             var(--p-purple-500);
  --color-chart-adx:             var(--p-red-500);
  --color-chart-atr:             var(--p-cyan-400);
  --color-chart-atr-decay:       var(--p-orange-500);
  --color-chart-vol-ratio:       var(--p-gray-500);
  --color-chart-accum-dist:      var(--p-yellow-500);

  --color-chart-entry-marker:    var(--p-green-500);
  --color-chart-partial-marker:  var(--p-yellow-500);
  --color-chart-exit-marker:     var(--p-red-500);
  --color-chart-stop-level:      var(--p-red-500);
  --color-chart-decision-dot:    var(--p-yellow-500);

  --color-chart-session-pre:     rgba(121, 85, 72, 0.08);
  --color-chart-session-after:   rgba(33, 150, 243, 0.08);
  --color-chart-position-profit: rgba(22, 163, 74, 0.05);
  --color-chart-position-loss:   rgba(220, 38, 38, 0.05);

  --color-chart-series-1: var(--p-blue-500);
  --color-chart-series-2: var(--p-green-500);
  --color-chart-series-3: var(--p-orange-500);
  --color-chart-series-4: var(--p-purple-500);
  --color-chart-series-5: var(--p-cyan-400);
  --color-chart-series-6: var(--p-pink-400);

  /* Typografie und Layout identisch zu Dark */
  --font-family-body:    var(--p-font-family-sans);
  --font-family-code:    var(--p-font-family-mono);
  --font-size-body:      var(--p-font-size-base);
  --font-size-label:     var(--p-font-size-sm);
  --font-size-caption:   var(--p-font-size-xs);
  --font-size-heading-1: var(--p-font-size-3xl);
  --font-size-heading-2: var(--p-font-size-2xl);
  --font-size-heading-3: var(--p-font-size-xl);
  --font-size-heading-4: var(--p-font-size-lg);
  --font-weight-body:    var(--p-font-weight-regular);
  --font-weight-label:   var(--p-font-weight-medium);
  --font-weight-heading: var(--p-font-weight-semibold);
  --line-height-body:    var(--p-line-height-normal);
  --line-height-heading: var(--p-line-height-tight);

  --layout-sidebar-width:   240px;
  --layout-header-height:   48px;
  --layout-content-padding: var(--p-space-6);
  --layout-panel-gap:       var(--p-space-4);
}
```

#### Ebene 3: Component Tokens (Bedarfsgesteuert)

Component Tokens bieten eine zusaetzliche Indirektionsstufe fuer Shared Components. Sie werden eingefuehrt, wenn eine Komponente mindestens zwei Varianten mit abweichender Token-Belegung hat.

**Einfuehrungskriterium:** Komponente hat >= 2 visuelle Varianten UND die Varianten unterscheiden sich in >= 2 Token-Belegungen. Ansonsten reichen direkte Semantic-Token-Referenzen.

**Ablage:** Component Tokens werden innerhalb der Shared-Component-Datei definiert (nicht in `design-system/tokens/`), da sie an die Komponente gebunden sind.

**Regel: Component Tokens referenzieren ausschliesslich Semantic Tokens** — keine Primitives. Falls Hover-/Active-Varianten benoetigt werden, muessen entsprechende Semantic Tokens ergaenzt werden (z.B. `--color-accent-hover`, `--color-negative-hover`). Damit bleibt die Theme-Semantik durchgaengig konsistent.

```css
/* shared/components/Button/Button.module.css */
.primary {
  --btn-bg: var(--color-accent);
  --btn-text: var(--color-text-inverse);
  --btn-hover-bg: var(--color-accent-hover);
}

.danger {
  --btn-bg: var(--color-negative);
  --btn-text: var(--color-text-inverse);
  --btn-hover-bg: var(--color-negative-hover);
}

.button {
  background: var(--btn-bg);
  color: var(--btn-text);
}
.button:hover {
  background: var(--btn-hover-bg);
}
```

> Die Semantic Tokens `--color-accent-hover` und `--color-negative-hover` werden in den Theme-Dateien definiert und koennen pro Theme unterschiedliche Primitive-Werte zuweisen.

### 2.3 Token-Governance

#### Naming-Konvention

| Ebene | Prefix | Schema | Beispiel |
|-------|--------|--------|----------|
| Primitive | `--p-` | `--p-{kategorie}-{stufe}` | `--p-gray-500`, `--p-space-4` |
| Semantic | `--color-`, `--font-`, `--layout-`, `--border-`, `--focus-`, `--opacity-` | `--{kategorie}-{element}-{variante}` | `--color-bg-surface`, `--font-size-body` |
| Component | `--{component}-` | `--{component}-{eigenschaft}` | `--btn-bg`, `--card-padding` |

#### Aenderungsregeln

- **Primitives** duerfen erweitert werden (neue Stufen/Farben hinzufuegen). Bestehende Werte nur mit Major-Version aendern
- **Semantics** sind der Contract fuer Feature-Code. Aenderungen an bestehenden Semantic Tokens sind Breaking Changes. Neue Tokens koennen jederzeit ergaenzt werden
- **Component Tokens** gehoeren der jeweiligen Shared Component und sind nicht Teil des oeffentlichen Contracts

#### Enforcement

- **Stylelint-Rule:** Verbietet `var(--p-*)` ausserhalb von `design-system/tokens/` und Component-Token-Definitionen
- **ESLint:** Verbietet Hex-Farbwerte in `.module.css` Dateien (ausser in `design-system/tokens/primitives.css`)
- **Code Review:** Token-Aenderungen erfordern Review des Design-System-Owners

### 2.4 Typografie-System

#### Schriftwahl

| Rolle | Schriftfamilie | Begruendung |
|-------|---------------|-------------|
| Body / UI | **Inter** | Optimiert fuer Screen-Lesbarkeit, klare Zahlenformen (Tabular Figures), exzellentes Kerning bei kleinen Groessen. Open Source, weit verbreitet |
| Code / Monospace | **JetBrains Mono** | Programmieroptimiert, klare Unterscheidung aehnlicher Zeichen (0/O, 1/l/I). Fuer Log-Anzeigen, Order-IDs, JSON |

#### Schriftgroessen-System (px-basiert)

Die Schriftgroessen sind **px-basiert** — nicht `rem` oder `em`. Begruendung: ODIN ist eine Desktop-only Anwendung fuer einen spezifischen Arbeitsplatz. Die Darstellung soll bei einer gegebenen Monitor-Aufloesung exakt voraussagbar sein, unabhaengig von Browser-Font-Settings.

| Token | Wert | Verwendung |
|-------|------|-----------|
| `--font-size-caption` | 11px | Timestamps, Sequence-IDs, Sekundaerinformation |
| `--font-size-label` | 12px | Labels, Tab-Titles, Badges, kleine Tabellen |
| `--font-size-body` | 14px | Fliesstext, Tabellenzellen, Listeneintraege |
| `--font-size-heading-4` | 18px | Panel-Ueberschriften, Section Headers |
| `--font-size-heading-3` | 20px | Page-Section-Titel |
| `--font-size-heading-2` | 24px | Page-Titel |
| `--font-size-heading-1` | 30px | Dashboard-Zahlen (grosse KPIs, Tages-P&L) |

#### Typografie-Presets

Fuer haeufig verwendete Kombinationen definieren wir CSS-Utility-Klassen:

```css
/* tokens/typography.css */

.typo-body {
  font-family: var(--font-family-body);
  font-size: var(--font-size-body);
  font-weight: var(--font-weight-body);
  line-height: var(--line-height-body);
  color: var(--color-text-primary);
}

.typo-label {
  font-family: var(--font-family-body);
  font-size: var(--font-size-label);
  font-weight: var(--font-weight-label);
  line-height: var(--line-height-tight);
  color: var(--color-text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.typo-caption {
  font-family: var(--font-family-body);
  font-size: var(--font-size-caption);
  font-weight: var(--font-weight-body);
  line-height: var(--line-height-normal);
  color: var(--color-text-tertiary);
}

.typo-heading-1 {
  font-family: var(--font-family-body);
  font-size: var(--font-size-heading-1);
  font-weight: var(--font-weight-heading);
  line-height: var(--line-height-heading);
  color: var(--color-text-primary);
}

.typo-code {
  font-family: var(--font-family-code);
  font-size: var(--font-size-label);
  font-weight: var(--font-weight-body);
  line-height: var(--line-height-normal);
}

.typo-number {
  font-family: var(--font-family-body);
  font-variant-numeric: tabular-nums;
  font-feature-settings: 'tnum' on;
}
```

> **Tabular Figures** (`font-variant-numeric: tabular-nums`) sind obligatorisch fuer alle numerischen Anzeigen (Kurse, P&L, KPI-Werte, Timestamps). Damit richten sich Zahlenkolonnen sauber aus, ohne Springen bei Updates.

### 2.5 Spacing-System (4px Base Grid)

Alle Abstands- und Groessenwerte basieren auf einem **4px-Grid**. Keine Magic Numbers.

| Token | Wert | Typische Verwendung |
|-------|------|-------------------|
| `--p-space-1` | 4px | Intra-Element (Icon-Text-Gap) |
| `--p-space-2` | 8px | Enge Nachbarschaft (Label-Value, Badge-Padding) |
| `--p-space-3` | 12px | Intra-Card-Padding (kompakte Bereiche) |
| `--p-space-4` | 16px | Standard-Padding, Panel-Gap |
| `--p-space-6` | 24px | Section-Padding, Content-Inset |
| `--p-space-8` | 32px | Groessere Abstaende zwischen Sections |
| `--p-space-12` | 48px | Page-Level-Separator |

### 2.6 JS-Token-Bridge (Chart-Konfiguration)

TradingView Lightweight Charts und Recharts erwarten konkrete Farbstrings in JavaScript — CSS Custom Properties (`var(--token)`) werden nicht direkt unterstuetzt. Fuer die Bruecke zwischen Token-System und Chart-Konfiguration:

```typescript
/* shared/utils/resolveToken.ts */

/**
 * Resolve a CSS Custom Property to its computed value.
 * Call after DOM mount and after theme change.
 */
function resolveToken(tokenName: string): string {
  return getComputedStyle(document.documentElement)
    .getPropertyValue(tokenName)
    .trim();
}

/**
 * Resolve a set of chart-relevant tokens.
 * Memoized per theme. Invalidated on theme change via EventBus.
 * Vollstaendige Liste aller Chart-Tokens — abgestimmt mit dem
 * Chart-Komponenten-Konzept (chart-component-concept.md §16).
 */
function resolveChartTokens(): ChartTokens {
  return {
    // Candlestick
    candleUp:        resolveToken('--color-chart-candle-up'),
    candleDown:      resolveToken('--color-chart-candle-down'),

    // Infrastruktur
    volume:          resolveToken('--color-chart-volume'),
    grid:            resolveToken('--color-chart-grid'),
    crosshair:       resolveToken('--color-chart-crosshair'),
    background:      resolveToken('--color-bg-surface'),
    textColor:       resolveToken('--color-text-primary'),

    // KPI-Overlays (Preis-Panel)
    ema9:            resolveToken('--color-chart-ema9'),
    ema21:           resolveToken('--color-chart-ema21'),
    ema50:           resolveToken('--color-chart-ema50'),
    ema100:          resolveToken('--color-chart-ema100'),
    vwap:            resolveToken('--color-chart-vwap'),
    bollinger:       resolveToken('--color-chart-bollinger'),

    // KPI-Overlays (Sub-Panels)
    rsi:             resolveToken('--color-chart-rsi'),
    adx:             resolveToken('--color-chart-adx'),
    atr:             resolveToken('--color-chart-atr'),
    atrDecay:        resolveToken('--color-chart-atr-decay'),
    volRatio:        resolveToken('--color-chart-vol-ratio'),
    accumDist:       resolveToken('--color-chart-accum-dist'),

    // Trading-Event-Marker
    entryMarker:     resolveToken('--color-chart-entry-marker'),
    partialMarker:   resolveToken('--color-chart-partial-marker'),
    exitMarker:      resolveToken('--color-chart-exit-marker'),
    stopLevel:       resolveToken('--color-chart-stop-level'),
    decisionDot:     resolveToken('--color-chart-decision-dot'),

    // Shading
    sessionPre:      resolveToken('--color-chart-session-pre'),
    sessionAfter:    resolveToken('--color-chart-session-after'),
    positionProfit:  resolveToken('--color-chart-position-profit'),
    positionLoss:    resolveToken('--color-chart-position-loss'),
  };
}
```

**Theme-Change-Handling:** Bei Theme-Wechsel wird ein Event (`global:theme-changed`) ueber den EventBus publiziert. Chart-Komponenten reagieren darauf und rufen `resolveChartTokens()` erneut auf, um die Chart-Instanz mit aktualisierten Farben zu konfigurieren. Details zur Theme-Integration im Chart: siehe [Chart-Komponenten-Konzept §16](chart-component-concept.md#16-farb-schema-und-design-token-integration).

### 2.7 Theme-Switching

```typescript
/* shared/hooks/useTheme.ts */

type Theme = 'dark' | 'light';

const STORAGE_KEY = 'odin-theme';
const DEFAULT_THEME: Theme = 'dark';

function useTheme(): { theme: Theme; toggleTheme: () => void } {
  // localStorage-backed state
  // Setzt data-theme Attribut auf document.documentElement
  // Dark ist Default (Trading-UI-Konvention)
}
```

- Theme-Praeferenz wird in `localStorage` persistiert
- Default: **Dark** (Trading-UI-Konvention, Guardrail §8). Dark ist Pflicht-Theme, Light ist optional
- Theme wird als `data-theme` Attribut auf `<html>` gesetzt
- Alle Semantic Tokens reagieren via `[data-theme='dark']` / `[data-theme='light']` Selektoren

**FOUC-Vermeidung (Flash of Wrong Theme):** Ein Inline-Script in `index.html` (vor React-Render) liest `localStorage` und setzt `data-theme` synchron:

```html
<!-- index.html, im <head> -->
<script>
  (function() {
    var theme = localStorage.getItem('odin-theme') || 'dark';
    document.documentElement.setAttribute('data-theme', theme);
  })();
</script>
```

Damit ist das korrekte Theme bereits gesetzt, bevor React hydratisiert und die erste Paint erfolgt.

### 2.7 Design System Verzeichnisstruktur

```
src/
  design-system/
    tokens/
      primitives.css          # Ebene 1: Rohwerte
      theme-dark.css          # Ebene 2: Dark Theme Semantics (Pflicht)
      theme-light.css         # Ebene 2: Light Theme Semantics (optional)
      typography.css          # Typografie-Presets
      index.css               # Import-Aggregator
    global/
      reset.css               # CSS Reset (modern-normalize)
      base.css                # html/body Base Styles
    index.css                 # Top-Level Design System Import
```

Diese Datei wird einmalig in `main.tsx` importiert — alle Design Tokens stehen danach global zur Verfuegung.

---

## 3. Domain-Driven Modulstruktur

### 3.1 Domaenen als architektonische Einheiten

Im Gegensatz zur bestehenden Feature-Struktur (Kap 9, Guardrail §2: `pipeline`, `chart`, `dashboard`, `controls`, `alerts`, `history`) fuehrt dieses Konzept eine **uebergeordnete Domaenenebene** ein. Features werden Domaenen zugeordnet. Eine Domaene ist eine fachlich geschlossene Einheit mit eigener Route, eigenem State und eigenen Features.

```
┌─────────────────────────────────────────────────────────────────────┐
│                           App Shell                                 │
│  ┌─────────┐  ┌─────────────────────────────────────────────────┐  │
│  │ Sidebar  │  │              Domain Content Area                │  │
│  │ (Nav)    │  │                                                 │  │
│  │ ┌──────┐ │  │  ┌─────────────────────────────────────────┐   │  │
│  │ │BckTst│ │  │  │         Active Domain Module            │   │  │
│  │ │TrdOps│ │  │  │                                         │   │  │
│  │ │Config│ │  │  │  ┌─────────┐  ┌─────────┐  ┌────────┐  │   │  │
│  │ │...   │ │  │  │  │Feature 1│  │Feature 2│  │Feature3│  │   │  │
│  │ └──────┘ │  │  │  └─────────┘  └─────────┘  └────────┘  │   │  │
│  └─────────┘  │  └─────────────────────────────────────────────┘ │  │
│               └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Domaenen-Blueprint

Jede Domaene folgt derselben Verzeichnisstruktur:

```
src/domains/{domain-name}/
  ├── components/              # Domaenen-spezifische Komponenten
  │   ├── {Component}.tsx
  │   └── {Component}.module.css
  ├── features/                # Feature-Module innerhalb der Domaene
  │   ├── {feature-name}/
  │   │   ├── components/      # Feature-lokale Komponenten
  │   │   ├── hooks/           # Feature-lokale Hooks
  │   │   ├── types/           # Feature-lokale Types
  │   │   └── index.ts         # Feature Public API
  │   └── ...
  ├── hooks/                   # Domaenen-spezifische Hooks
  ├── types/                   # Domaenen-spezifische Types
  ├── store/                   # Domaenen-eigener State Slice
  │   └── {domain}Store.ts
  ├── routes.tsx               # Domaenen-internes Routing
  └── index.ts                 # Domain Public API (Barrel Export)
```

**Regeln fuer Domaenen:**

1. Jede Domaene exportiert ueber `index.ts` nur das, was die App-Shell braucht: die Route-Konfiguration und ggf. ein Navigation-Item
2. Domaene-zu-Domaene-Imports sind **verboten**
3. Innerhalb einer Domaene duerfen Features den `store/` der Domaene nutzen
4. Feature-zu-Feature-Imports innerhalb einer Domaene sind **nur ueber die Feature Public API (`index.ts`)** erlaubt — keine direkten Imports in Feature-Interna
5. Jede Domaene definiert ihren eigenen State Slice (zustandslos ist auch erlaubt)

### 3.3 Vollstaendige Verzeichnisstruktur

```
odin-frontend/
├── public/
├── src/
│   ├── design-system/               # Design System (§2)
│   │   ├── tokens/
│   │   ├── global/
│   │   └── index.css
│   │
│   ├── shared/                       # Shared Infrastructure (domain-agnostisch)
│   │   ├── api/                      # SSE-Client, REST-Client, Reconnect-Logik
│   │   │   ├── sseClient.ts
│   │   │   ├── restClient.ts
│   │   │   └── types.ts
│   │   ├── components/               # Shared UI Components (Design System Bausteine)
│   │   │   ├── Button/
│   │   │   ├── Badge/
│   │   │   ├── Card/
│   │   │   ├── ConfirmDialog/
│   │   │   ├── DataTable/
│   │   │   ├── StatusIndicator/
│   │   │   ├── AlertBanner/
│   │   │   ├── LoadingSpinner/
│   │   │   ├── EmptyState/
│   │   │   └── index.ts
│   │   ├── hooks/                    # Shared Hooks
│   │   │   ├── useAuth.ts
│   │   │   ├── useTheme.ts
│   │   │   ├── useEventSource.ts
│   │   │   └── index.ts
│   │   ├── types/                    # Globale Types (Enums, DTOs, Event-Types)
│   │   │   ├── pipelineState.ts
│   │   │   ├── sseEvents.ts
│   │   │   ├── tradingRun.ts
│   │   │   ├── common.ts
│   │   │   └── index.ts
│   │   ├── store/                    # Globaler Store + Event-Bus
│   │   │   ├── globalStore.ts
│   │   │   ├── eventBus.ts
│   │   │   └── index.ts
│   │   └── utils/                    # Utilities
│   │       ├── formatCurrency.ts
│   │       ├── formatTimestamp.ts
│   │       ├── formatPercent.ts
│   │       └── index.ts
│   │
│   ├── domains/                      # Fachliche Domaenen
│   │   ├── backtesting/              # Domaene: Backtesting (§7)
│   │   │   ├── components/
│   │   │   ├── features/
│   │   │   │   ├── backtest-launcher/
│   │   │   │   ├── backtest-results/
│   │   │   │   └── backtest-comparison/
│   │   │   ├── hooks/
│   │   │   ├── types/
│   │   │   ├── store/
│   │   │   ├── routes.tsx
│   │   │   └── index.ts
│   │   │
│   │   ├── trading-operations/       # Domaene: Trading Operations (§8)
│   │   │   ├── components/
│   │   │   ├── features/
│   │   │   │   ├── live-dashboard/
│   │   │   │   ├── pipeline-monitor/
│   │   │   │   ├── chart-view/
│   │   │   │   └── controls/
│   │   │   ├── hooks/
│   │   │   ├── types/
│   │   │   ├── store/
│   │   │   ├── routes.tsx
│   │   │   └── index.ts
│   │   │
│   │   └── ... (weitere Domaenen)
│   │
│   ├── app/                          # App Shell
│   │   ├── App.tsx                   # Root Component
│   │   ├── AppLayout.tsx             # Layout: Header + Sidebar + Content
│   │   ├── AppRouter.tsx             # Routing-Aggregator
│   │   ├── AppProviders.tsx          # Context Providers Stack
│   │   └── Sidebar.tsx               # Navigation
│   │
│   └── main.tsx                      # Entry Point
│
├── package.json
├── tsconfig.json
├── vite.config.ts
└── vitest.config.ts
```

### 3.4 Abhaengigkeitsregeln (erweitert)

```
┌────────────────────────────────────────────────────────────────┐
│  app/                                                          │
│  ├── importiert: domains/**/index.ts (Route-Config)            │
│  ├── importiert: shared/** (Infrastruktur)                     │
│  └── importiert: design-system/index.css (einmalig)            │
├────────────────────────────────────────────────────────────────┤
│  domains/{domain}/                                             │
│  ├── importiert: shared/** (Infrastruktur)                     │
│  ├── importiert NICHT: domains/{andere-domain}/**  [VERBOTEN]  │
│  └── importiert NICHT: app/**  [VERBOTEN]                      │
├────────────────────────────────────────────────────────────────┤
│  shared/                                                       │
│  ├── importiert NICHT: domains/**  [VERBOTEN]                  │
│  └── importiert NICHT: app/**  [VERBOTEN]                      │
├────────────────────────────────────────────────────────────────┤
│  design-system/                                                │
│  └── Reine CSS-Dateien, keine TS-Imports                       │
└────────────────────────────────────────────────────────────────┘
```

**ESLint-Enforcement:** Import-Restriktionen werden via `eslint-plugin-boundaries` erzwungen. Gegenueber `import/no-restricted-paths` bietet dieses Plugin Element-Typ-basierte Konfiguration, die automatisch fuer neue Domaenen funktioniert.

```javascript
// .eslintrc.js (Auszug)
module.exports = {
  plugins: ['boundaries'],
  settings: {
    'boundaries/elements': [
      { type: 'app', pattern: 'src/app/*' },
      { type: 'shared', pattern: 'src/shared/*' },
      { type: 'domain', pattern: 'src/domains/*', capture: ['domainName'] },
      { type: 'design-system', pattern: 'src/design-system/*' },
    ],
  },
  rules: {
    'boundaries/element-types': ['error', {
      default: 'disallow',
      rules: [
        // app darf shared und domains importieren
        { from: 'app', allow: ['shared', 'domain'] },
        // domains duerfen shared und sich selbst importieren (same-domain)
        { from: 'domain', allow: ['shared', ['domain', { domainName: '${from.domainName}' }]] },
        // shared darf weder domains noch app importieren
        { from: 'shared', allow: [] },
      ],
    }],
    // Feature-Imports innerhalb einer Domain nur ueber index.ts (Public API)
    'boundaries/entry-point': ['error', {
      default: 'disallow',
      rules: [
        // Innerhalb einer Domain: Features nur ueber index.ts erreichbar
        { target: 'domain', allow: ['index.ts', 'index.tsx'] },
      ],
    }],
  },
};
```

> **Hinweis:** Die `domainName`-Capture-Variable stellt sicher, dass Domain A nur auf sich selbst zugreifen darf, nicht auf Domain B. Die `entry-point`-Regel erzwingt, dass Feature-Imports innerhalb einer Domain ueber die Feature Public API (`index.ts`) laufen.

---

## 4. State Management & Event-Architektur

### 4.1 State-Architektur Ueberblick

```
┌──────────────────────────────────────────────────────────┐
│                     React Context                         │
│              AuthContext, ThemeContext                      │
├──────────────────────────────────────────────────────────┤
│                    Zustand Store                          │
│  ┌──────────────┐ ┌───────────────┐ ┌────────────────┐  │
│  │ Global Slice │ │Backtesting    │ │ TradingOps     │  │
│  │ (SSE, Risk,  │ │Slice          │ │ Slice          │  │
│  │  Alerts,     │ │               │ │                │  │
│  │  Connection) │ │               │ │                │  │
│  └──────┬───────┘ └───────┬───────┘ └────────┬───────┘  │
│         │                 │                   │          │
│         └─────────────────┼───────────────────┘          │
│                           │                              │
│                    ┌──────┴──────┐                        │
│                    │  Event Bus  │                        │
│                    └─────────────┘                        │
└──────────────────────────────────────────────────────────┘
```

**Single Source of Truth fuer Connection-Status:** `connectionStatus` lebt ausschliesslich im Global Store (Zustand), nicht in einem separaten React Context. Context bleibt fuer Auth und Theme reserviert — beides Singletons mit seltenen Updates.

### 4.2 Store-Technologie: Zustand

**Entscheidung:** Zustand als State Management Library.

**Begruendung:**

| Kriterium | React Context | Redux Toolkit | Zustand |
|-----------|--------------|---------------|---------|
| Boilerplate | Minimal | Hoch | Minimal |
| Slice-Pattern | Nein (eigener Code) | Ja (native) | Ja (via Slices) |
| Re-Render | Alle Consumers | Selector-basiert | Selector-basiert |
| DevTools | Nein | Ja | Ja (Middleware) |
| Bundle Size | 0 KB | ~12 KB | ~1.5 KB |
| SSE-Integration | Umstaendlich | Gut | Gut |
| Lernkurve | Niedrig | Mittel | Niedrig |

Zustand bietet Selector-basiertes Re-Render (kritisch fuer Echtzeit-Updates per SSE), Slice-kompatibilitaet fuer Domain-Slices, minimale Bundle Size und geringste Boilerplate. Redux ist fuer diesen Use Case Over-Engineering. React Context allein fuehrt zu unnoetigem Re-Rendering bei haeufigen SSE-Updates.

> **Guardrail-Update noetig:** Der bestehende Frontend-Guardrail (§5) sagt "Kein Redux / Zustand". Diese Entscheidung wird hier revidiert. Siehe §1.4 fuer Details.

### 4.3 Store-Struktur

#### Global Store Scope Rule

Der Global Store enthaelt **ausschliesslich cross-domain-relevante Daten**:

| Im Global Store | Begruendung |
|----------------|-------------|
| `connectionStatus` | Infrastruktur, relevant fuer alle Domaenen (SSE-Verbindung) |
| `accountRisk` | Globaler Risk-Status, relevant fuer Trading Ops und Dashboard |
| `killSwitchActive` | Globaler Kill-Switch, beeinflusst alle Domaenen |
| `alerts` | System-Alerts, domaenen-uebergreifend |
| `pipelines` | Pipeline-Snapshots aus SSE, konsumiert von Trading Ops |

**Regel:** Wenn Daten nur von einer Domaene gelesen und geschrieben werden, gehoeren sie in den Domain Store — nicht in den Global Store. Der Global Store ist kein "God Store". Neue Felder im Global Store erfordern die Begruendung, warum sie cross-domain relevant sind.

#### Global Slice (SSE-Daten, Account Risk, Alerts, Connection)

```typescript
/* shared/store/globalStore.ts */

interface GlobalState {
  // Connection
  readonly connectionStatus: ConnectionStatus;

  // Account Risk (aus SSE Global Stream)
  readonly accountRisk: AccountRiskState | null;

  // Alerts
  readonly alerts: ReadonlyArray<Alert>;

  // Kill-Switch
  readonly killSwitchActive: boolean;

  // Pipeline States (alle aktiven Pipelines, aus SSE Instrument Streams)
  readonly pipelines: ReadonlyMap<string, PipelineSnapshot>;

  // Actions
  updateAccountRisk: (risk: AccountRiskState) => void;
  addAlert: (alert: Alert) => void;
  acknowledgeAlert: (alertId: string) => void;
  updatePipelineSnapshot: (instrumentId: string, updater: Partial<PipelineSnapshot>) => void;
  replacePipelineSnapshot: (instrumentId: string, snapshot: PipelineSnapshot) => void;
  setKillSwitchActive: (active: boolean) => void;
  setConnectionStatus: (status: ConnectionStatus) => void;
}
```

> **Update-Pattern:** `updatePipelineSnapshot` fuer partielle Updates (mergt in bestehenden Snapshot), `replacePipelineSnapshot` fuer vollstaendige Snapshot-Replacements (z.B. nach Reconnect). Store-intern wird immer ein neues Objekt erzeugt (immutable).

#### Domain Slice (Beispiel: Backtesting)

```typescript
/* domains/backtesting/store/backtestingStore.ts */

interface BacktestingState {
  // Backtest-Liste
  readonly backtests: ReadonlyArray<BacktestSummary>;
  readonly selectedBacktestId: string | null;
  readonly backtestDetail: BacktestDetail | null;

  // Laufender Backtest
  readonly activeBacktest: ActiveBacktestStatus | null;

  // Actions
  setBacktests: (backtests: ReadonlyArray<BacktestSummary>) => void;
  selectBacktest: (backtestId: string) => void;
  setBacktestDetail: (detail: BacktestDetail) => void;
  setActiveBacktest: (status: ActiveBacktestStatus | null) => void;
}
```

### 4.4 Event-Bus fuer Cross-Domain-Kommunikation

Domaenen duerfen nicht direkt aufeinander zugreifen. Wenn eine Aktion in Domaene A eine Reaktion in Domaene B ausloesen soll, geschieht dies ueber einen zentralen Event-Bus.

#### Design: Typsicheres Event-Registry via Declaration Merging

Der Event-Bus kombiniert **statische Typsicherheit** mit **dezentraler Event-Definition**. Domaenen erweitern ein zentrales `EventMap`-Interface per TypeScript Declaration Merging — ohne den `shared/`-Code zu aendern.

```typescript
/* shared/store/eventBus.ts */

/**
 * Central event map interface. Domains extend this via Declaration Merging.
 * Do NOT add events here directly — each domain extends in its own types/events.ts.
 */
interface DomainEventMap {
  // Extended by domains via declaration merging
}

type EventType = keyof DomainEventMap;

interface EventBus {
  /**
   * Publish a typed event. TypeScript enforces correct payload for each event type.
   */
  publish: <K extends EventType>(type: K, payload: DomainEventMap[K]) => void;

  /**
   * Subscribe to a typed event. Returns unsubscribe function.
   */
  subscribe: <K extends EventType>(
    type: K,
    handler: (payload: DomainEventMap[K]) => void
  ) => () => void;
}

/**
 * Create the singleton EventBus.
 * - Handlers called synchronously, each within try/catch (failure isolation).
 * - Subscriber rule: handlers MUST be O(1). Heavy work (toasts, navigation,
 *   REST calls) must be deferred via queueMicrotask or setTimeout.
 */
function createEventBus(): EventBus {
  const handlers = new Map<string, Set<(payload: unknown) => void>>();

  return {
    publish(type, payload) {
      const typeHandlers = handlers.get(type as string);
      if (!typeHandlers) return;
      for (const handler of typeHandlers) {
        try {
          handler(payload);
        } catch (error) {
          console.error(`[EventBus] Handler error for '${type as string}':`, error);
        }
      }
    },
    subscribe(type, handler) {
      const key = type as string;
      if (!handlers.has(key)) {
        handlers.set(key, new Set());
      }
      handlers.get(key)!.add(handler as (payload: unknown) => void);
      return () => {
        handlers.get(key)?.delete(handler as (payload: unknown) => void);
      };
    },
  };
}
```

#### Domain-lokale Event-Definitionen (via Declaration Merging)

Jede Domaene erweitert das `DomainEventMap`-Interface lokal:

```typescript
/* domains/backtesting/types/events.ts */

declare module '@shared/store/eventBus' {
  interface DomainEventMap {
    'backtest:completed': { readonly backtestId: string; readonly summary: BacktestSummary };
    'backtest:started': { readonly backtestId: string };
  }
}
```

```typescript
/* domains/trading-operations/types/events.ts */

declare module '@shared/store/eventBus' {
  interface DomainEventMap {
    'trading:kill-switch-activated': { readonly timestamp: string };
    'trading:pipeline-state-changed': { readonly instrumentId: string; readonly newState: PipelineState };
  }
}
```

**Vorteile:**

- Eine neue Domaene erweitert das Interface lokal, ohne `shared/store/eventBus.ts` oder andere Domaenen zu aendern
- TypeScript prueft zur Kompilierzeit, ob `publish('backtest:completed', payload)` den korrekten Payload-Typ hat
- Der Event-Bus-Code selbst bleibt unveraendert — nur das TypeScript Type System wird erweitert

#### Subscriber-Pattern

```typescript
/* Beispiel: Trading Operations lauscht auf Backtest-Completion */

function useTradingOpsEventSubscriptions(): void {
  const eventBus = useEventBus();

  useEffect(() => {
    const unsubscribe = eventBus.subscribe('backtest:completed', (payload) => {
      // payload ist statisch typisiert: { backtestId: string; summary: BacktestSummary }
      // Subscriber-Regel: O(1) work only. Defer heavy work:
      queueMicrotask(() => {
        // Notification anzeigen, State aktualisieren
      });
    });
    return unsubscribe;
  }, [eventBus]);
}
```

### 4.5 SSE-Integration mit dem Store

SSE-Daten fliessen ueber dedizierte Hooks direkt in den Global Store.

> **Sonderfall Chart-Komponente:** Die IntraDayChart-Komponente (siehe [Chart-Komponenten-Konzept](chart-component-concept.md)) nutzt `store.subscribe()` statt render-basierter Selektoren, um SSE-Daten imperativ an die TradingView Lightweight Charts API weiterzugeben — ohne React-Render-Zyklus. Die SSE-Daten-Invariante (Daten fliessen ueber den Store) wird eingehalten, aber der Update-Pfad umgeht React fuer Performance bei 1-Sekunden-Updates. Details: Chart-Konzept §6.

#### SSE-Technikentscheidung: Fetch-basiertes Streaming

**Entscheidung:** Der SSE-Client nutzt **`fetch()` mit `ReadableStream`** (nicht die native `EventSource` API).

**Begruendung:** Die native `EventSource` API unterstuetzt keine Custom Headers (insbesondere `Authorization` fuer HTTP Basic Auth, Kap 9 §3). Fuer ODIN ist Authentifizierung Pflicht. `fetch()` mit `ReadableStream` und manueller SSE-Line-Parsing ermoeglicht volle Kontrolle ueber Headers, Auth-Token-Refresh und `Last-Event-ID`.

```typescript
/* shared/api/sseClient.ts — Technische Basis */

// Verbindung aufbauen mit Auth-Header
const response = await fetch(url, {
  headers: {
    'Accept': 'text/event-stream',
    'Authorization': `Basic ${credentials}`,
    'Last-Event-ID': lastEventId,
  },
});
const reader = response.body!.getReader();
const decoder = new TextDecoder();
// Manuelles SSE-Line-Parsing: event, data, id Felder
```

#### SSE-Client Contract (`shared/api/sseClient.ts`)

Der SSE-Client implementiert die Resync-Mechanik aus Kap 9 §3:

| Mechanik | Implementierung |
|----------|----------------|
| **Last-Event-ID** | Jeder empfangene Event-ID wird getracked. Bei Reconnect wird `Last-Event-ID` Header gesendet |
| **Reconnect** | Exponential Backoff: 1s, 2s, 4s, 8s, max 30s (Kap 9 §3) |
| **Event-ID-Reset-Erkennung** | Wenn nach Reconnect `eventId < lastKnownEventId`: Backend wurde restartet. Client erzwingt Full-Snapshot-Refresh (alle Daten via REST nachladen) |
| **Duplikat-Idempotenz** | Empfangene Event-IDs werden in einem Set (max 1000 Eintraege, FIFO) gehalten. Duplikate werden ignoriert |
| **Heartbeat** | Bei fehlendem Event innerhalb von 30s: Verbindung als gestorben betrachten, Reconnect einleiten |

#### Hook-Beispiele

```typescript
/* shared/hooks/useGlobalStream.ts */

function useGlobalStream(): void {
  const store = useGlobalStore;

  useEventSource('/api/v1/stream/global', {
    onEvent: (event: GlobalSseEvent) => {
      switch (event.type) {
        case 'risk-update':
          store.getState().updateAccountRisk(event.payload);
          break;
        case 'alert':
          store.getState().addAlert(event.payload);
          break;
        case 'pnl':
          // P&L-Updates verarbeiten
          break;
      }
    },
    onConnectionChange: (status: ConnectionStatus) => {
      store.getState().setConnectionStatus(status);
    },
  });
}
```

```typescript
/* shared/hooks/usePipelineStream.ts */

function usePipelineStream(instrumentId: string): void {
  const store = useGlobalStore;
  const eventBus = useEventBus();

  useEventSource(`/api/v1/stream/instruments/${instrumentId}`, {
    onEvent: (event: InstrumentSseEvent) => {
      switch (event.type) {
        case 'snapshot':
          store.getState().replacePipelineSnapshot(instrumentId, event.payload);
          break;
        case 'state-change':
          store.getState().updatePipelineSnapshot(instrumentId, {
            state: event.payload.newState,
          });
          eventBus.publish('trading:pipeline-state-changed', {
            instrumentId,
            newState: event.payload.newState,
          });
          break;
      }
    },
  });
}
```

### 4.6 React Context (nur fuer Singletons)

React Context bleibt fuer wirkliche Singletons, die selten wechseln:

| Context | Inhalt | Update-Frequenz |
|---------|--------|----------------|
| `AuthContext` | User, Role (OPERATOR/VIEWER), Token | Nur bei Login/Logout |
| `ThemeContext` | Theme (dark/light), toggleTheme | Selten (manueller Toggle) |
| `EventBusContext` | EventBus-Instanz | Nie (Singleton) |

Fuer alle Daten mit hoher Update-Frequenz (SSE-Streams, Connection-Status, Alerts) wird **ausschliesslich** Zustand verwendet.

---

## 5. Routing-Konzept

### 5.1 Domain-basiertes Routing

Jede Domaene besitzt einen eigenen URL-Namespace:

```
/                              -> Redirect zu /trading (oder Dashboard)
/backtesting                   -> Backtesting-Domaene
/backtesting/new               -> Neuen Backtest konfigurieren
/backtesting/:backtestId       -> Backtest-Detail
/backtesting/:backtestId/chart -> Chart-Analyse eines Backtest-Runs
/trading                       -> Trading Operations Dashboard
/trading/pipeline/:instrumentId -> Pipeline-Detail
/trading/chart/:instrumentId   -> Live-Chart
/config                        -> Konfiguration (zukuenftig)
/history                       -> History/Reporting (zukuenftig)
```

### 5.2 Route-Registration

Domaenen exportieren ihre Route-Konfiguration. Die App-Shell aggregiert sie:

```typescript
/* domains/backtesting/routes.tsx */

export const backtestingRoutes: RouteObject = {
  path: 'backtesting',
  children: [
    { index: true, element: <BacktestListPage /> },
    { path: 'new', element: <BacktestLauncherPage /> },
    { path: ':backtestId', element: <BacktestDetailPage /> },
    { path: ':backtestId/chart', element: <BacktestChartPage /> },
  ],
};
```

```typescript
/* app/AppRouter.tsx */

import { backtestingRoutes } from '@domains/backtesting';
import { tradingRoutes } from '@domains/trading-operations';

const router = createBrowserRouter([
  {
    element: <AppLayout />,
    children: [
      backtestingRoutes,
      tradingRoutes,
      // weitere Domaenen hier anhaengen
      { path: '*', element: <Navigate to="/trading" replace /> },
    ],
  },
]);
```

### 5.3 Lazy Loading (pro Domaene)

Domaenen werden per `React.lazy()` geladen — so wird der initiale Bundle nur die aktive Domaene enthalten:

```typescript
const BacktestingDomain = lazy(() => import('@domains/backtesting'));
const TradingDomain = lazy(() => import('@domains/trading-operations'));
```

---

## 6. Komponentenhierarchie

### 6.1 Vier Ebenen

```
Layout Components (App Shell)
  └── Domain Components (Domain-spezifisch)
       └── Feature Components (Feature-spezifisch)
            └── Shared UI Components (Design System)
```

| Ebene | Ort | Beispiele | Verantwortung |
|-------|-----|-----------|--------------|
| **Layout** | `app/` | `AppLayout`, `Sidebar`, `Header`, `ContentArea` | Globales Layout, Navigation, Provider-Stack |
| **Domain** | `domains/{d}/components/` | `BacktestSummaryCard`, `PipelineOverviewPanel` | Domaenen-spezifische zusammengesetzte Ansichten |
| **Feature** | `domains/{d}/features/{f}/components/` | `BacktestForm`, `ResultsTable`, `ChartOverlay` | Feature-lokale Teilkomponenten |
| **Shared UI** | `shared/components/` | `Button`, `Badge`, `Card`, `DataTable`, `ConfirmDialog` | Wiederverwendbare, domaenen-agnostische UI-Bausteine |

### 6.2 Shared UI Component Library

| Komponente | Zweck | Varianten |
|-----------|-------|-----------|
| `Button` | Primaere Aktion | `primary`, `secondary`, `danger`, `ghost`, Sizes: `sm`, `md` |
| `Badge` | Status-Anzeige | Farbvarianten per Token, Dot-Variante |
| `Card` | Container mit optionalem Header/Footer | `surface`, `elevated` |
| `ConfirmDialog` | Bestaetigung fuer destruktive Aktionen | `danger`, `warning` |
| `DataTable` | Tabellarische Daten | Sortierbar, paginierbar, mit Slots. Fuer grosse Datensaetze: Row-Virtualisierung via TanStack Virtual |
| `StatusIndicator` | Pipeline-State / Connection-State | Dot + Label, Farbcodierung via Tokens |
| `AlertBanner` | Inline-Alerts | `info`, `warning`, `critical`, mit Dismiss |
| `LoadingSpinner` | Lade-Indikator | Inline, Overlay |
| `EmptyState` | Leere Listen/Ergebnisse | Icon + Beschreibung + optionale Action |
| `Tooltip` | Kontext-Information | Positioned, Delay |
| `Tabs` | Tab-Navigation innerhalb einer Seite | Horizontal |
| `Tag` | Kategorisierungs-Label | Farbvarianten |

---

## 7. Referenz-Blueprint: Domaene Backtesting

### 7.1 Fachlicher Kontext

Die Backtesting-Domaene deckt den vollstaendigen Backtest-Lifecycle ab: Konfiguration, Ausfuehrung, Ergebnis-Analyse und Vergleich. Sie konsumiert die Backend-APIs aus WP-02 bis WP-06 (Requirements-Dokument).

### 7.2 Verzeichnisstruktur

```
src/domains/backtesting/
  ├── components/
  │   ├── BacktestSummaryCard.tsx
  │   ├── BacktestSummaryCard.module.css
  │   ├── BacktestStatusBadge.tsx
  │   └── PerformanceMetricGrid.tsx
  ├── features/
  │   ├── backtest-launcher/
  │   │   ├── components/
  │   │   │   ├── BacktestForm.tsx
  │   │   │   ├── InstrumentSelector.tsx
  │   │   │   ├── DateRangeSelector.tsx
  │   │   │   ├── ParameterSetSelector.tsx
  │   │   │   └── DataAvailabilityCheck.tsx
  │   │   ├── hooks/
  │   │   │   ├── useBacktestSubmit.ts
  │   │   │   └── useDataAvailability.ts
  │   │   ├── types/
  │   │   │   └── backtestRequest.ts
  │   │   └── index.ts
  │   ├── backtest-results/
  │   │   ├── components/
  │   │   │   ├── ResultsSummaryPanel.tsx
  │   │   │   ├── DailyResultsTable.tsx
  │   │   │   ├── TradeListTable.tsx
  │   │   │   ├── EquityCurveChart.tsx
  │   │   │   └── BacktestChartView.tsx
  │   │   ├── hooks/
  │   │   │   ├── useBacktestReport.ts
  │   │   │   ├── useBacktestTrades.ts
  │   │   │   └── useBacktestBars.ts
  │   │   ├── types/
  │   │   │   └── backtestReport.ts
  │   │   └── index.ts
  │   └── backtest-comparison/
  │       ├── components/
  │       │   ├── ComparisonSelector.tsx
  │       │   └── ComparisonTable.tsx
  │       ├── hooks/
  │       │   └── useBacktestComparison.ts
  │       └── index.ts
  ├── hooks/
  │   ├── useBacktestList.ts
  │   └── useBacktestPolling.ts
  ├── types/
  │   ├── backtest.ts
  │   ├── backtestStatus.ts
  │   └── events.ts
  ├── store/
  │   └── backtestingStore.ts
  ├── routes.tsx
  └── index.ts
```

### 7.3 State Slice

```typescript
interface BacktestingState {
  // Liste aller Backtests
  readonly backtests: ReadonlyArray<BacktestSummary>;
  readonly backtestsLoading: boolean;
  readonly backtestsTotalPages: number;
  readonly backtestsCurrentPage: number;

  // Selektierter Backtest
  readonly selectedBacktestId: string | null;
  readonly backtestDetail: BacktestDetail | null;
  readonly backtestDetailLoading: boolean;

  // Aktiver (laufender) Backtest
  readonly activeBacktest: ActiveBacktestStatus | null;

  // Filter
  readonly filters: BacktestFilters;

  // Actions
  setBacktests: (backtests: ReadonlyArray<BacktestSummary>, totalPages: number) => void;
  setCurrentPage: (page: number) => void;
  selectBacktest: (backtestId: string | null) => void;
  setBacktestDetail: (detail: BacktestDetail | null) => void;
  setActiveBacktest: (status: ActiveBacktestStatus | null) => void;
  updateActiveBacktestProgress: (progress: BacktestProgress) => void;
  setFilters: (filters: Partial<BacktestFilters>) => void;
}
```

### 7.4 Pages und Routing

| Route | Page | Beschreibung |
|-------|------|-------------|
| `/backtesting` | `BacktestListPage` | Paginierte Liste, Filter, Status-Badges, Quick-Actions |
| `/backtesting/new` | `BacktestLauncherPage` | Formular: Instrumente, Zeitraum, Intervall, ParameterSet, Datenverfuegbarkeit |
| `/backtesting/:id` | `BacktestDetailPage` | Tabs: Summary, Daily Results, Trades, Chart |
| `/backtesting/:id/chart` | `BacktestChartPage` | Vollbild-Chart mit KPI-Overlays, Entry/Exit-Marker, Decision-Points |

### 7.5 Cross-Domain-Events (publiziert)

| Event | Payload | Typischer Konsument |
|-------|---------|-------------------|
| `backtest:completed` | `{ backtestId, summary }` | Trading Ops (Notification) |
| `backtest:started` | `{ backtestId }` | Global (Toast) |

---

## 8. Referenz-Blueprint: Domaene Trading Operations

### 8.1 Fachlicher Kontext

Die Trading-Operations-Domaene deckt den operativen Handelstag ab: Live-Dashboard, Pipeline-Monitoring, Chart-Ansicht und manuelle Controls (Kill-Switch, Pause/Resume). Sie konsumiert SSE-Streams und REST-Endpoints aus Kap 9.

### 8.2 Verzeichnisstruktur

```
src/domains/trading-operations/
  ├── components/
  │   ├── TradingDayHeader.tsx
  │   ├── TradingDayHeader.module.css
  │   ├── SimulationModeBanner.tsx
  │   └── AccountRiskPanel.tsx
  ├── features/
  │   ├── live-dashboard/
  │   │   ├── components/
  │   │   │   ├── DashboardGrid.tsx
  │   │   │   ├── PnlSummaryCard.tsx
  │   │   │   ├── ExposureCard.tsx
  │   │   │   ├── TradeCountCard.tsx
  │   │   │   └── AlertFeed.tsx
  │   │   ├── hooks/
  │   │   │   └── useDashboardData.ts
  │   │   └── index.ts
  │   ├── pipeline-monitor/
  │   │   ├── components/
  │   │   │   ├── PipelineStatusPanel.tsx
  │   │   │   ├── PipelineStateIndicator.tsx
  │   │   │   ├── PositionSummary.tsx
  │   │   │   ├── OmsStateDisplay.tsx
  │   │   │   ├── LlmStatusDisplay.tsx
  │   │   │   └── TrancheOverview.tsx
  │   │   ├── hooks/
  │   │   │   └── usePipelineData.ts
  │   │   └── index.ts
  │   ├── chart-view/
  │   │   ├── components/
  │   │   │   ├── TradingChart.tsx
  │   │   │   ├── ChartTimeframeSelector.tsx
  │   │   │   ├── IndicatorOverlay.tsx
  │   │   │   ├── EntryExitMarkers.tsx
  │   │   │   ├── PnlChart.tsx
  │   │   │   └── QuantScoreChart.tsx
  │   │   ├── hooks/
  │   │   │   ├── useChartBars.ts
  │   │   │   ├── useChartIndicators.ts
  │   │   │   └── useChartTrades.ts
  │   │   └── index.ts
  │   └── controls/
  │       ├── components/
  │       │   ├── KillSwitchButton.tsx
  │       │   ├── PipelinePauseResumeButton.tsx
  │       │   └── ControlsPanel.tsx
  │       ├── hooks/
  │       │   ├── useKillSwitch.ts
  │       │   └── usePipelineControl.ts
  │       └── index.ts
  ├── hooks/
  │   ├── useGlobalStream.ts
  │   ├── usePipelineStream.ts
  │   └── useInstrumentList.ts
  ├── types/
  │   ├── tradingDay.ts
  │   ├── pipeline.ts
  │   ├── control.ts
  │   └── events.ts
  ├── store/
  │   └── tradingOpsStore.ts
  ├── routes.tsx
  └── index.ts
```

### 8.3 State Slice

```typescript
interface TradingOpsState {
  // Aktive Instrumente
  readonly instruments: ReadonlyArray<InstrumentInfo>;

  // Aktuelle Runs
  readonly currentRuns: ReadonlyArray<TradingRunSummary>;

  // Selektierte Pipeline (fuer Detail-View)
  readonly selectedInstrumentId: string | null;

  // Chart-State
  readonly chartTimeframe: ChartTimeframe;

  // Actions
  setInstruments: (instruments: ReadonlyArray<InstrumentInfo>) => void;
  setCurrentRuns: (runs: ReadonlyArray<TradingRunSummary>) => void;
  selectInstrument: (instrumentId: string | null) => void;
  setChartTimeframe: (tf: ChartTimeframe) => void;
}
```

### 8.4 Pages und Routing

| Route | Page | Beschreibung |
|-------|------|-------------|
| `/trading` | `TradingDashboardPage` | Global-Dashboard: P&L, Exposure, Risk, Pipeline-Uebersicht, Alerts, Kill-Switch |
| `/trading/pipeline/:instrumentId` | `PipelineDetailPage` | Pipeline-Detail: State, Position, Tranchen, OMS, LLM-Status |
| `/trading/chart/:instrumentId` | `LiveChartPage` | Echtzeit-Chart mit Indikatoren, Entry/Exit-Marker, P&L-Verlauf |

### 8.5 SSE-Integration

Die Trading-Operations-Domaene ist der primaere Konsument der SSE-Streams:

```typescript
/* In TradingDashboardPage einmalig initialisiert */
function TradingDashboardPage(): ReactElement {
  useGlobalStream();  // Verbindet mit /api/v1/stream/global -> Global Store

  const instruments = useTradingOpsStore((s) => s.instruments);

  // Pro aktivem Instrument einen Pipeline-Stream
  return (
    <>
      {instruments.map((inst) => (
        <PipelineStreamConnector key={inst.instrumentId} instrumentId={inst.instrumentId} />
      ))}
      <DashboardGrid />
    </>
  );
}
```

> **Hinweis zu Stream-Anzahl:** ODIN unterstuetzt in v1 max. 2-3 parallele Instrumente (Kap 0). Die Anzahl der SSE-Verbindungen (1 global + 2-3 instrument) bleibt damit weit unter dem Browser-Limit von 6 gleichzeitigen HTTP-Verbindungen pro Origin. Sollte die Instrumentzahl signifikant steigen, waere ein Multiplexing-Endpoint zu evaluieren.

### 8.6 Cross-Domain-Events (publiziert)

| Event | Payload | Typischer Konsument |
|-------|---------|-------------------|
| `trading:kill-switch-activated` | `{ timestamp }` | Backtesting (Info) |
| `trading:pipeline-state-changed` | `{ instrumentId, newState }` | Global (Notification) |

---

## 9. Technologieentscheidungen

| Bereich | Entscheidung | Begruendung |
|---------|-------------|-------------|
| **Framework** | React 18+ | Vorgabe (Kap 0). Mature, grosses Ecosystem |
| **Sprache** | TypeScript (strict) | Vorgabe (Guardrail). Type Safety, bessere IDE-Unterstuetzung |
| **Build Tool** | Vite + SWC | Vorgabe (Guardrail §9). Schneller Build, HMR |
| **Routing** | React Router v6+ | Marktstandard, Nested Routes, Lazy Loading |
| **State Management** | Zustand | Leichtgewichtig, Selector-basiert, SSE-kompatibel (§4.2) |
| **Charting** | TradingView Lightweight Charts | Vorgabe (Kap 9 §4.1). Trading-optimiert, Candlestick-Charts. Detaillierte Chart-Architektur: [Chart-Komponenten-Konzept](chart-component-concept.md) |
| **Supplementary Charts** | Recharts | Vorgabe (Kap 9 §4.1). P&L-Verlauf, Score-History |
| **Styling** | CSS Modules + CSS Custom Properties | Vorgabe (Guardrail §8). Scoped Styles, Design Tokens |
| **Fonts** | Inter (Sans), JetBrains Mono (Code) | Optimiert fuer Screen-Lesbarkeit und Zahlenform (§2.4) |
| **Testing** | Vitest + React Testing Library | Vorgabe (Guardrail §10). Schnell, React-nativ |
| **Linting** | ESLint + Prettier + `eslint-plugin-boundaries` | Guardrail §9, erweitert um Architecture Boundary Enforcement |
| **Event-Bus** | Eigener Pub/Sub mit Error Isolation | Lean-Prinzip. Synchron mit try/catch je Handler (§4.4) |
| **HTTP Client** | Fetch API (typisierter Wrapper) | Kein Axios noetig. Shared REST-Client (Guardrail §6) |
| **Table Virtualization** | TanStack Virtual (bei Bedarf) | Fuer DataTable bei grossen Datensaetzen (Trade-Listen, Decision-Logs) |
| **Import-Aliase** | `@domains/`, `@shared/`, `@app/`, `@design-system/` | Guardrail §9, erweitert um `@domains/` und `@design-system/` |

---

## 10. Performance & Realtime-Guardrails

Fuer eine Trading-UI mit SSE-basierten Echtzeit-Updates gelten spezifische Performance-Regeln:

### 10.1 SSE-Update-Verarbeitung

| Regel | Details |
|-------|---------|
| **Store-Updates via `getState()`** | SSE-Event-Handler muessen `store.getState().action(...)` verwenden, nicht per Selector geholt und ueber Closure geschlossen. Vermeidet Stale Closures |
| **Immutable Updates** | Store-Actions erzeugen immer neue Objekte/Maps. Keine Mutation bestehender Referenzen. Zustand erkennt Changes via `Object.is()` |
| **Selector-Granularitaet** | Jede Komponente selektiert nur die minimal benoetigten Felder. Nicht `useGlobalStore()` (ganzer Store), sondern `useGlobalStore((s) => s.accountRisk)` |
| **Shallow-Compare** | Fuer Selektoren die Objekte zurueckgeben: `useGlobalStore(selector, shallow)` verwenden |

### 10.2 Chart-Performance

Detaillierte Chart-Architektur und Performance-Ueberlegungen: siehe [Chart-Komponenten-Konzept](chart-component-concept.md) §15.

| Regel | Details |
|-------|---------|
| **TradingView LW Charts** | Update-Methoden (`update()`, `setData()`) ausserhalb des React Render-Cycles aufrufen (via `store.subscribe()` + imperativ). Kein Selector-basiertes Re-Render fuer Chart-Daten |
| **Recharts** | Daten-Updates per Zustand-Selector; Recharts rendert automatisch bei Prop-Change |
| **Keine unnoetige Re-Renders** | Chart-Wrapper-Komponenten mit `React.memo` nur bei nachgewiesenem Problem (Lean-Prinzip) |

### 10.3 Tabellen-Performance

| Regel | Details |
|-------|---------|
| **Serverseitige Paginierung** | Alle Listen-Endpoints nutzen die Backend-Paginierung (REQ-WEB-004). Client laedt nur eine Page |
| **Row-Virtualisierung** | Fuer Tabellen mit potenziell > 100 Zeilen (Trade-Listen, Decision-Logs): TanStack Virtual einsetzen |
| **Stabile Keys** | Domain-IDs (`backtestId`, `runId`, `tradeId`) als React `key` — keine Array-Indices |

### 10.4 Accessibility-Minimum (A11y)

ODIN ist eine Desktop-only Operator-Anwendung, keine oeffentliche Web-App. Trotzdem gelten grundlegende A11y-Regeln:

| Anforderung | Regel |
|-------------|-------|
| **Keyboard-Navigation** | Alle interaktiven Elemente (Buttons, Links, Dialogs, Tabs) muessen per Tastatur erreichbar und bedienbar sein |
| **Focus-Ring** | Sichtbarer Focus-Indicator auf allen interaktiven Elementen. Tokens: `--focus-ring-color`, `--focus-ring-width`, `--focus-ring-offset` |
| **Farbkontrast** | Text auf Hintergrund: mindestens 4.5:1 (WCAG AA). Signal-Token-Paarungen (z.B. `--color-positive` auf `--color-bg-surface`) muessen dieses Kriterium erfuellen |
| **Rot-Gruen-Blindheit** | P&L und Pipeline-States duerfen sich nicht ausschliesslich durch Rot/Gruen unterscheiden. Zusaetzliche Indikatoren: Icons, Symbole, Textlabels |
| **Reduced Motion** | `prefers-reduced-motion: reduce` deaktiviert Animationen und Transitions. Token-basierte Transitions (`--p-transition-*`) werden in einer `@media (prefers-reduced-motion: reduce)`-Rule auf `0ms` gesetzt |
| **ARIA-Patterns** | Shared Components (Dialog, DataTable, Tabs, Tooltip) implementieren die entsprechenden WAI-ARIA-Patterns |

---

## 11. Erweiterbarkeitsgarantien

### 11.1 Checkliste: Neue Domaene hinzufuegen

Wenn eine neue fachliche Domaene (z.B. "Configuration", "Analytics", "Optimization") hinzugefuegt werden soll:

- [ ] **Verzeichnis erstellen:** `src/domains/{domain-name}/` mit dem Blueprint aus §3.2
- [ ] **Route definieren:** `routes.tsx` mit den Domaenen-Routen
- [ ] **Store Slice erstellen:** `store/{domain}Store.ts` (falls State benoetigt)
- [ ] **Barrel Export:** `index.ts` exportiert Route-Config und ggf. NavItem
- [ ] **Route registrieren:** In `app/AppRouter.tsx` die Route importieren und einhaengen
- [ ] **Navigation:** In `app/Sidebar.tsx` den Nav-Eintrag hinzufuegen
- [ ] **Lazy Loading:** `React.lazy(() => import('@domains/{domain-name}'))`
- [ ] **Events definieren:** Domaenen-lokale Event-Types in `types/events.ts` (kein zentraler Union-Type noetig)
- [ ] **Tests:** Unit-Tests fuer Hooks und Store-Logic
- [ ] **Build pruefen:** `npm run build` + `npm run lint` erfolgreich

**Aenderungen ausserhalb des neuen Domain-Ordners:** Nur `app/AppRouter.tsx` und `app/Sidebar.tsx`. Keine Aenderungen in `shared/`, keine Aenderungen in anderen Domaenen, keine ESLint-Config-Aenderung (dank `eslint-plugin-boundaries` mit Pattern-Matching).

### 11.2 Architektur-Invarianten

| Invariante | Pruefung |
|-----------|----------|
| Keine Cross-Domain-Imports | `eslint-plugin-boundaries` (CI-enforced) |
| Feature-Imports nur ueber Public API (`index.ts`) | `eslint-plugin-boundaries` entry-point Rule |
| Design Tokens als einzige Farbquelle | Stylelint (keine Hex-Werte in `.module.css` ausser `tokens/`) |
| Primitives nicht direkt referenziert | Stylelint (kein `var(--p-*)` ausserhalb `tokens/`) |
| Component Tokens referenzieren nur Semantic Tokens | Code Review (keine `var(--p-*)` in Component Tokens) |
| Alle numerischen Anzeigen mit Tabular Figures | Code Review |
| SSE-Daten nur ueber Store, nicht ueber lokalen State | Code Review + Hook-Konvention |
| Global Store nur fuer cross-domain Daten | Code Review + Scope Rule (§4.3) |
| Event-Bus-Subscriber: O(1) work only | Code Review + Konvention |
| Jede Domaene hat `index.ts`, `routes.tsx`, `types/` | CI-Script oder Verzeichnis-Lint |

### 11.3 Migrations-Hinweis

Die bestehende Feature-Struktur aus dem Guardrail (§2: `features/pipeline`, `features/chart`, etc.) wird in die neue Domaenen-Struktur migriert:

| Alt (Guardrail §2) | Neu (Domains) |
|--------------------|---------------|
| `features/pipeline/` | `domains/trading-operations/features/pipeline-monitor/` |
| `features/chart/` | `domains/trading-operations/features/chart-view/` |
| `features/dashboard/` | `domains/trading-operations/features/live-dashboard/` |
| `features/controls/` | `domains/trading-operations/features/controls/` |
| `features/alerts/` | Aufgeteilt: Shared Component (`AlertBanner`) + Domain Feature |
| `features/history/` | Aufgeteilt: `domains/backtesting/features/backtest-results/` + ggf. eigene History-Domaene |

---

## Anhang A: Import-Alias Konfiguration

```json
/* tsconfig.json (Auszug) */
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@app/*": ["src/app/*"],
      "@domains/*": ["src/domains/*"],
      "@shared/*": ["src/shared/*"],
      "@design-system/*": ["src/design-system/*"]
    }
  }
}
```

```typescript
/* vite.config.ts (Auszug) */
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@app': resolve(__dirname, 'src/app'),
      '@domains': resolve(__dirname, 'src/domains'),
      '@shared': resolve(__dirname, 'src/shared'),
      '@design-system': resolve(__dirname, 'src/design-system'),
    },
  },
});
```

## Anhang B: Referenz-Dokumente

| Dokument | Pfad |
|----------|------|
| Chart-Komponenten-Konzept | `frontend/architecture/chart-component-concept.md` |
| Kap 9: React-Frontend | `frontend/architecture/09-frontend.md` |
| Frontend-Guardrail | `frontend/guardrails/frontend.md` |
| Web Application Requirements | `frontend/requirements/web-application-requirements.md` |
| Kap 0: Systemuebersicht | `architecture/00-system-overview.md` |
