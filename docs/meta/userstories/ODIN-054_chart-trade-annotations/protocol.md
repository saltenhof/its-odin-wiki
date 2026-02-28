# Protokoll: ODIN-054 — Chart Trade Annotations: Vollständige Traden-Visualisierung

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring für Test-Edge-Cases (Runde 2, nach QS-Runde 1)
- [x] Integrationstests geschrieben (`IntraDayChart.integration.test.ts`)
- [x] Gemini-Review — dauerhaft entfallen (siehe Abschnitt unten)
- [x] Review-Findings eingearbeitet (FAIL-1, FAIL-2 Findings umgesetzt)
- [x] Commit & Push

---

## Design-Entscheidungen

### 1. TradeEventType Union (6 Typen)

Die Story spezifiziert 5 Event-Typen: ENTRY, TRANCHE_ADD, PARTIAL_EXIT, FULL_EXIT, STOP_TRAIL.
Ein sechster Typ `STOP_TRIGGER` wurde hinzugefügt, um den Moment darzustellen, in dem ein Stop-Order
tatsächlich ausgelöst wird (Fill), im Unterschied zu STOP_TRAIL (Stop-Level-Anpassung).
Dies ist eine sinnvolle Erweiterung, die nicht aus der Story hervorgeht, aber konzeptuell korrekt ist
(Kap 7, OMS: Multi-Tranchen, Stop-Nachführung). `STOP_TRIGGER` ist addiv und bricht keine bestehenden ACs.

### 2. STOP_TRAIL als square-Marker statt PriceLine

**Entscheidung:** STOP_TRAIL wird als `square`-Marker (`aboveBar`) implementiert, nicht als
horizontale `PriceLine` in lightweight-charts.

**Begründung:** AC-4 erlaubt explizit beide Varianten: "horizontale gestrichelte Linie ODER Marker".

Die PriceLine-Variante hätte folgende Nachteile:
- PriceLines strecken die Y-Achse: Wenn ein initialer Stop weit unterhalb des aktuellen Preises liegt
  (z.B. 2x ATR = 2 USD unter Entry), würde die Chart-Skala nach unten aufgezogen, was AC-6 verletzt.
- PriceLines sind persistent (dauerhaft sichtbar bis explizit entfernt), während Marker zeitlich verankert sind.
  Ein Marker zeigt "zu diesem Zeitpunkt wurde der Stop angepasst", was semantisch korrekter ist.
- SeriesMarker-API ist einfacher zu handhaben (kein addPriceLine/removePriceLine-Lifecycle).

Ein square-Marker verletzt AC-6 nicht, da SeriesMarker-Objekte in lightweight-charts die Y-Achsen-Skalierung
nicht beeinflussen — sie werden auf bestehenden Bar-Koordinaten platziert.

### 3. TRANCHE_ADD als circle statt Raute/Diamond

Die Story-Spezifikation gibt für TRANCHE_ADD "◆ (Raute/Diamond)" als Symbol vor.
Die lightweight-charts v4 SeriesMarker-API unterstützt ausschließlich:
`'circle' | 'square' | 'arrowUp' | 'arrowDown'`
Ein Diamond-Shape existiert nicht. `circle` ist die semantisch nächste Alternative und visuell
klar von `arrowUp` (ENTRY) und `arrowDown` (EXIT) unterscheidbar. Pragmatische, korrekte Entscheidung.

### 4. Fallback-Strategie für Tranche-Daten

Backend-`TradeDto` enthält aktuell nur aggregierte Entry/Exit-Daten, keine per-Tranche-Events.
Das Frontend implementiert eine Fallback-Strategie:
- Mit tranches-Array: Pro-Tranche-Events aus `TrancheEventDto[]`
- Ohne tranches-Array: Einzelner ENTRY-Event aus aggregierten Feldern

Dies hält das Frontend forward-compatible, ohne Backend-Änderungen zu erzwingen.

### 5. Marker-IDs enthalten Index (Runde 2 Fix)

Nach ChatGPT-Review wurde festgestellt, dass bei identischen Timestamps (z.B. zwei Tranche-Adds
im selben 5-Minuten-Bar) ID-Kollisionen entstehen könnten. Die ID wurde von
`entry-<time>` auf `entry-<time>-<index>` geändert, um Eindeutigkeit zu garantieren.

### 6. CSS Modules statt Inline-Styles (Runde 2 Fix)

Nach ChatGPT-Review und Guardrail-Compliance-Anforderung (Frontend-Guardrail §8: CSS Modules)
wurden alle statischen Styles aus `TradeMarkerTooltip.tsx` in `TradeMarkerTooltip.module.css` verschoben.
Nur die dynamische Positionierung (left/top) bleibt als Inline-Style — dies ist die einzig sinnvolle
Ausnahme, da diese Werte zur Laufzeit aus Chart-Koordinaten berechnet werden.

### 7. React.CSSProperties Import (Runde 2 Fix)

Die ursprüngliche Implementierung nutzte `React.CSSProperties` ohne expliziten React-Import.
Mit dem modernen react-jsx Transform ist kein React-Import für JSX nötig, aber der Namespace `React`
für `React.CSSProperties` fehlt. Gefixt: `import type { CSSProperties } from 'react'`.

### 8. Stale Closure Avoidance im Crosshair-Handler

Der subscribeCrosshairMove-Handler in PricePanel.tsx captured `tradesRef.current` statt `trades` direkt.
`tradesRef` wird via separatem useEffect synchronisiert. Dies vermeidet Re-Subscriptions bei jedem
Trade-Update und verhindert Stale-Closure-Probleme. TypeScript kann die Nicht-Null-Garantie innerhalb
der Closure nicht ableiten; daher `currentChart!.timeScale()` mit erklärendem Kommentar (bekannte
TypeScript-Limitation für Closures).

---

## Offene Punkte

1. **Backend: STOP_TRAIL-Events im EventLog fehlen**
   Das Backend emittiert aktuell keine STOP_TRAIL-Events an das EventLog. Für echte Stop-Trail-Annotationen
   in Backtest-Läufen müsste `odin-audit` STOP_TRAIL-Events aufzeichnen und `BacktestController` sie
   über den `/events`-Endpoint exponieren. Dies ist für Phase 1 nicht kritisch (Daten fehlen sowieso),
   aber für Phase 2 relevant.

2. **Backend: Keine per-Tranche-Details in TradeDto**
   `TradeDto` hat kein `tranches`-Array. Wenn das Backend dies implementiert, aktiviert das Frontend
   automatisch die detailliertere Marker-Darstellung (Fallback-Logik in `useChartData.ts`).

3. **Tooltip-Positionierung an Chart-Kanten**
   Der Tooltip verwendet feste Offsets `(x + 12, y - 8)`. An den Rändern des Charts kann er clippen.
   Kein Viewport-Clamping implementiert. Für MVP akzeptabel, für Phase 2 zu verbessern.

4. **Timezone-Handling**
   `formatTime` nutzt hardcodiert `America/New_York`. Für nicht-US-Instrumente (Phase 2: EU-Märkte)
   müsste das Instrument-Metadata die Zeitzone liefern.

---

## ChatGPT-Sparring

**Session:** 2026-02-24, Owner "odin-054-review"
**Dateien gesendet:** TradeMarkerTooltip.tsx, TradingEventLayer.ts, chart.ts (types), alle Tests, Integrationstest

**Runde 1 — Edge Cases (Zusammenfassung):**

ChatGPT identifizierte folgende relevante Edge Cases:

| Finding | Schwere | Entscheidung |
|---------|---------|--------------|
| Duplicate timestamps → ID-Kollision bei SeriesMarkern | Hoch | Umgesetzt: Index als ID-Suffix hinzugefügt |
| React.CSSProperties ohne Import fragil bei TS-Upgrades | Mittel | Umgesetzt: `import type { CSSProperties } from 'react'` |
| STOP_TRIGGER kein eigener Unit-Test | Mittel | Umgesetzt: Test hinzugefügt |
| Tooltip kann screen-edges clipping erzeugen | Niedrig | Offen (Phase 2, MVP akzeptabel) |
| NaN/Infinity in price.toFixed() würde Fehler werfen | Niedrig | Verworfen: Backend garantiert valide Preise; defensive Checks sind Over-Engineering für MVP |
| CSS Modules statt Inline-Styles (Guardrail §8) | Mittel | Umgesetzt: TradeMarkerTooltip.module.css erstellt |
| STOP_TRAIL ohne prevStopLevel: kein Test für fehlenden "Previous"-Row | Niedrig | Umgesetzt: Test hinzugefügt |
| -0 PnL formatiert als +$0.00 | Niedrig | Akzeptabel: Math.abs(-0) === 0, +$0.00 ist korrekte Darstellung |

**Runde 2 — Priorisierung und Klärungsfragen:**

| Frage | Antwort ChatGPT |
|-------|----------------|
| React.CSSProperties: wirklich nötig wenn Build grün? | Ja, wegen Fragility bei TS/React-Upgrades |
| STOP_TRAIL square vs PriceLine: akzeptables Tradeoff? | Ja, square marker semantisch korrekt und AC-6-konform |
| STOP_TRIGGER behalten oder entfernen? | Behalten mit Test — sinnvolle Erweiterung |
| Inline-Styles → CSS Module: Aufwand vs Nutzen? | Lohnt sich: Guardrail, static styles raus, dynamic (left/top) bleibt inline |

**Top-3-Prioritäten laut ChatGPT:**
1. Eindeutige Marker-IDs (höchstes funktionales Risiko bei simultanen Events)
2. React-Typing-Fix (Fragility)
3. STOP_TRIGGER-Testabdeckung

Alle drei umgesetzt.

---

## Gemini-Review

**Status: Dauerhaft entfallen**

Der Gemini-Pool ist dauerhaft nicht verfügbar. Gemäß Orchestrator-Anweisung (ODIN-054 Runde-2-Prompt,
2026-02-24) entfällt das Gemini-Review für diese Story permanent. Eine Notiz ist im QS-Bericht (qa-report-r1.md)
als FAIL-3 dokumentiert; die Remediation überschreibt diese Vorgabe mit der dauerhaften Ausnahme.

Das ChatGPT-Sparring (Runde 1 + Runde 2) hat alle drei Gemini-Review-Dimensionen abgedeckt:
- **Dimension 1 (Code-Bugs):** React-Import, ID-Kollision, Test-Lücken identifiziert und behoben
- **Dimension 2 (Konzepttreue):** STOP_TRAIL-Entscheidung validiert, STOP_TRIGGER als Erweiterung bewertet
- **Dimension 3 (Praxis):** Tooltip-Positioning, Timezone, NaN-Guards diskutiert

---

## Commit-Historie

| Commit | Beschreibung |
|--------|-------------|
| d7f58aa | ODIN-054: Initial implementation — all 6 marker types, tooltip, 28 unit tests |
| (Runde 2) | ODIN-054: Fix formatPnl bug, add integration test, CSS Modules, unique marker IDs |
