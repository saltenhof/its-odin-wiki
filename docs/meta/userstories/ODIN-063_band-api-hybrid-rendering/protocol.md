# Protokoll: ODIN-063 -- Band API + Hybrid UI Rendering

## Working State
- [x] Initiale Implementierung (Backend: ZoneType Enum + SrZoneDto)
- [x] Initiale Implementierung (Frontend: LEVEL-Rendering)
- [x] Initiale Implementierung (Frontend: BAND-Rendering)
- [x] Unit-Tests geschrieben (Backend)
- [x] Unit-Tests geschrieben (Frontend)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (Backend)
- [x] Integrationstests geschrieben (Frontend)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Playwright E2E-Tests mit Screenshot-Beweisen
- [ ] Commit & Push

## Design-Entscheidungen

### 1. ZoneType als enum in odin-brain (nicht in odin-api)
- `ZoneType` liegt in `de.its.odin.brain.sr` (nicht in odin-api)
- Begruendung: ZoneType ist Brain-interne Logik (ConsolidationBandDetector setzt BAND).
  odin-api kennt kein ZoneType — der Typ wird als String `"LEVEL"` / `"BAND"` ueber das DTO kommuniziert.
  Eine API-Enum wuerde eine direkte Abhaengigkeit von odin-api auf brain erfordern.

### 2. SrZoneDto: String statt Enum fuer zoneType
- `zone.zoneType().name()` serialisiert den Enum als String "LEVEL" oder "BAND"
- Begruendung: Vermeidet strenge Enum-Deserialisierung auf dem Frontend; zukunftssicher fuer neue ZoneTypes
- Frontend akzeptiert beliebige Strings und faellt bei unbekannten Werten auf LEVEL-Rendering zurueck

### 3. BAND-Fill via Baseline-Series (nicht Histogram)
- Urspruenglicher Ansatz: Histogram-Series mit synthetischen Timestamps 1 und 2
- Problem (ChatGPT-Sparring): Timestamps 1/2 liegen ausserhalb des sichtbaren Chartbereichs
  → Fill wurde nie gerendert
- Fix: `chart.addBaselineSeries()` mit `baseValue.price = bottom` und Wert = `top`
- Zwei Datenpunkte bei `barTimes[0]` und `barTimes[last]` anchorn den Fill an den geladenen Bar-Bereich
- `autoscaleInfoProvider: () => null` verhindert, dass das Band die Price-Scale-Autoskalierung stoert

### 4. Degenerate Band (band=0): nur POC-Linie
- `BAND_MIN_WIDTH = 1e-8` Guard: wenn effectiveBand <= BAND_MIN_WIDTH, nur POC-Linie rendern
- Verhindert Duplikat-Linien am gleichen Preis (bottom == top == center)
- Begruendung (ChatGPT-Sparring): drei identische gestrichelte Linien am gleichen Preis sehen kaputt aus

### 5. latestBarsRef-Pattern in useSrLines (nach Gemini-Review)
- Urspruenglicher Ansatz: `bars` als React-Dependency in `useEffect` → `setZones` bei jedem neuen Bar
- Problem (Gemini-Review): `setZones` ruft `removeAllVisuals()` auf, d.h. jeder neue Bar
  zerstoert und recreiert alle S/R-Overlays (Flicker, Memory-Churn, Canvas-Overhead)
- Fix: `latestBarsRef` haelt immer den aktuellen `bars`-Zustand ohne Effect-Re-Runs auszuloesen
- S/R-Zonen sind per Session statisch — kein Grund, sie bei jedem Candle neu zu zeichnen
- `applyZones()` liest aus `latestBarsRef`, wodurch async-Callbacks in fetch-Promises nie stale Daten sehen

### 6. barTimes optional: BAND ohne Fill wenn keine bars vorhanden
- Wenn `barTimes` leer oder undefined: BAND rendert nur Edge-Linien + POC, kein Fill
- Begruendung: S/R-Zonen koennen theoretisch ankommen bevor Bars geladen sind (Race Condition)
  Lieber kein Fill als einen Crash oder falschen Fill

### 7. currentPrice optional: LEVEL-Color-Fallback
- Wenn `currentPrice` undefined: LEVEL-Zonen verwenden `COLOR_UNKNOWN_SIDE` (gruenes Fallback)
- Begruendung: Beim ersten Render vor Bars-Load koennen Zonen noch keine korrekte Support/Resistance-Farbe bestimmen
- In der Praxis selten relevant: latestBarsRef stellt sicher, dass currentPrice bei fetch-Resolve stimmt

## Offene Punkte
- Playwright E2E-Test: Screenshot-Beweis fuer LEVEL + BAND Rendering (wird in QS-Agent durchgefuehrt)

## ChatGPT-Sparring

### Session 1: Test-Edge-Cases und Band-Fill-Ansatz

**Ausgangsfrage:** Korrekte Edge-Cases fuer BAND-Rendering + welcher LWC-Ansatz fuer horizontalen Fill?

**Finding 1 (kritisch): Histogram-Ansatz kaputt**
- Problem: `addHistogramSeries` mit Timestamps 1 und 2 produziert keinen sichtbaren Fill
  (die Timestamps liegen ausserhalb des Chartbereichs)
- Fix: `addBaselineSeries` mit echten Bar-Timestamps als Anchor
- Umgesetzt: Kompletter Rewrite der `createBandFillSeries`-Funktion auf Baseline-Ansatz

**Finding 2 (mittel): band=0 Degenerate Case**
- Problem: BAND mit `band=0` erzeugt 3 identische Linien am gleichen Preis
- Fix: `BAND_MIN_WIDTH = 1e-8` Guard mit Early-Return auf nur POC-Linie
- Umgesetzt: `renderBandZone()` prueft `effectiveBand <= BAND_MIN_WIDTH` vor Linien-Erstellung

**Finding 3 (minor): color ohne currentPrice**
- Problem: Ohne currentPrice bekommen alle LEVEL-Zonen dieselbe Farbe (kein Support/Resistance-Unterschied)
- Fix: `setZones()` akzeptiert optionalen `currentPrice`-Parameter; `resolveLevelColor()` nutzt ihn
- Umgesetzt: Parameter-Erweiterung von `setZones(zones, currentPrice?, barTimes?)`

## Gemini-Review

### Session 1: Initialer Code-Review

**Dimension 1 — Code-Qualitaet**
- ZoneType enum: korrekt
- SrZone-Mutation: akzeptabel (SrZone ist bereits ein mutable POJO, konsistent mit bestehendem Design)
- SrZoneDto: `zone.zoneType().name()` als String korrekt fuer REST-APIs (+1)
- Kritischer Bug gefunden: `useSrLines.ts` rief `setZones(zones)` ohne `currentPrice` und `barTimes` auf
  → LEVEL-Farben immer Fallback-Gruen, BAND-Fill nie gerendert

**Dimension 2 — Konzepttreue**
- Alle Farb-Konstanten und Opacity-Werte korrekt implementiert
- Graceful Degradation vorhanden
- Degenerate Band abgefangen
- Label-Format korrekt
- Bewertung: Konzepttreu (+1)

**Dimension 3 — Praxis**
- Baseline-Series + 2 Datenpunkte: wird korrekten horizontalen Fill rendern
  Hinweis: Fill endet bei barTimes[0] und barTimes[last] — Scroll ausserhalb macht Fill unsichtbar (erwartetes Verhalten)
- `autoscaleInfoProvider: () => null`: korrekte Methode (+1)
- `lineWidth=1` statt 0: LWC enforced min=1; Workaround via transparente Linienfarben korrekt (+1)
- `destroy()` try/catch: pragmatisch und standard-praxis bei LWC React-Wrappern (+1)

### Session 2: useSrLines Performance-Review

**Kritisches Finding (Session 2):**
- Urspruengliche Version mit `bars` als Effect-Dependency fuehrte zu `setZones()` bei jedem Bar-Update
- `setZones()` → `removeAllVisuals()` → komplette DOM/Canvas-Rekonstruktion aller S/R-Overlays
- Bei Live-Trading: mehrfach pro Sekunde (Tick-Updates) wenn `bars` sich aendert
- Fix: `latestBarsRef`-Pattern eliminiert `bars` aus Effect-Dependencies voellig
- Ergebnis: S/R-Overlays werden nur noch erstellt bei (a) Handle-Create, (b) Fetch-Completion
- ESLint-disable-Kommentare entfernt; saubere Effect-Dependencies ohne stale Closures

**Weitere Findings (alle umgesetzt):**
- Race-Condition-Analyse: latestBarsRef loest Stale-Closure bei async fetch-Promise
- Lean-Entscheidung: keine bar-boundary-Crossing-Detektion fuer Color-Updates (YAGNI)

## Playwright E2E-Tests

5 Tests bestanden. Testdatei: `e2e/sr-lines-rendering.spec.ts`

| # | Test | Screenshot | Beschreibung |
|---|------|-----------|-------------|
| 1 | API zoneType-Feld | — | Alle Zonen haben gueltigen zoneType ("LEVEL" oder "BAND") |
| 2 | Kein Rendering-Fehler | — | Keine Console-Errors waehrend S/R-Rendering |
| 3 | LEVEL-Rendering | `odin-063-sr-level-rendering.png` | Preis-Panel sichtbar, S/R-Linien geladen |
| 4 | Keine Warnings | — | Keine useSrLines-Warnings (fetch-Fehler etc.) |
| 5 | Vollstaendiges Hybrid-Rendering | `odin-063-sr-hybrid-full-chart.png` | Kompletter Chart mit BAND + LEVEL Zonen |

**Befund aus Screenshot `odin-063-sr-hybrid-full-chart.png` (IREN, 2026-02-23):**
- BAND-Zone sichtbar: Grosses oranges halbtransparentes Rechteck (Consol. $40.95--$43.06)
- LEVEL-Zonen sichtbar: Horizontale Linien bei $41.33 (CLUSTER,VWAP, blau), $40.47/$40.32 (OR, gruen), $38.96 (OR, gruen)
- VWAP-Linie in Blau (#2196F3): korrekt
- Support-Linien in Gruen (#0EB35B): korrekt
- Band-Label-Format "Consol. $X.XX--$Y.YY": korrekt
