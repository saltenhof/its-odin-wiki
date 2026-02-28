# Story: ODIN-054 — Chart Trade Annotations: Vollständige Traden-Visualisierung

## Metadaten

| Feld | Wert |
|------|------|
| **Titel** | Chart Trade Annotations: Vollständige Traden-Visualisierung |
| **Modul** | odin-frontend (primär), odin-backtest / odin-execution (Backend-Endpoint-Erweiterung falls nötig) |
| **Phase** | 1 |
| **Abhängigkeiten** | Keine (kann unabhängig entwickelt werden; Chart-Komponente und TradingEventLayer existieren bereits) |
| **Geschätzter Umfang** | M |

---

## Kontext

Die Chart-Komponente (`IntraDayChart`) zeigt bereits Candlesticks, VWAP, EMAs und andere technische Indikatoren. Eine `TradingEventLayer` ist bereits implementiert und zeigt den initialen Entry und den finalen Exit als Marker. Für ein wirkliches Verständnis eines Backtest-Laufs braucht der Nutzer aber die vollständige Trade-Chronologie auf dem Chart: Nachkäufe (Tranche-Adds), Teilverkäufe (Partial Exits), Stop-Anpassungen (Trailing Stop Moves). Nur so kann der Nutzer ablesen, wie ODIN in einer konkreten Situation gehandelt hat.

Ziel dieser Story: Die bestehende `TradingEventLayer` um alle relevanten Trade-Ereignisse erweitern, mit visuell klaren und unterscheidbaren Symbolen/Markern, und einem informativen Tooltip bei Hover.

---

## Scope

**In Scope:**
- Erweitern der `TradingEventLayer` um folgende Event-Typen:
  - **ENTRY (Erstpositionierung, T1):** Grüner Aufwärtspfeil — bereits vorhanden, ggf. verbessern
  - **TRANCHE_ADD (Nachkauf T2/T3/...):** Grünes Plus-Symbol (◆ oder +) — klar unterscheidbar vom Erst-Entry
  - **PARTIAL_EXIT (Teilverkauf):** Orangefarbenes/gelbes Abwärtspfeil (↓) — gibt Tranche-PnL im Tooltip
  - **FULL_EXIT (Vollständiger Ausstieg):** Roter Abwärtspfeil — bereits vorhanden, ggf. verbessern
  - **STOP_TRAIL (Stop-Anpassung nach oben):** Horizontale Linie oder kleines Schloss-Symbol (🔒) — zeigt neue Stop-Ebene
- Tooltip bei Hover auf jeden Marker: zeigt Event-Typ, Preis, Uhrzeit, Stück, realisierten PnL (wenn vorhanden), Exit-Grund (wenn vorhanden)
- Marker-Daten kommen aus dem bestehenden `/api/v1/runs/{runId}/trades`-Endpoint (ggf. Erweiterung nötig, falls Tranche-Events dort nicht vollständig sind) ODER aus dem Event-Log (`/api/v1/backtests/{id}/events?eventType=TRANCHE_ADD`)
- Live-Trading-Modus: gleiche Annotations auch für laufende Trades (wenn Run-ID vorhanden)

**Out of Scope:**
- Neue Chart-Panels (kein neues Panel für Trade-Events)
- Interaktive Bearbeitung von Stops über den Chart (kein Drag&Drop)
- LLM-Begründungs-Anzeige im Chart (das gehört in den Event-Log, nicht in den Chart)
- Replay/Animation des Trade-Verlaufs

---

## Akzeptanzkriterien

- [ ] **AC-1:** Nachkäufe (TRANCHE_ADD) erscheinen als grünes, klar von Erst-Entry unterscheidbares Symbol auf dem Chart, an der korrekten Zeitposition und Preislinie.
- [ ] **AC-2:** Teilverkäufe (PARTIAL_EXIT) erscheinen als orangefarbenes/gelbes Symbol an Zeitposition und Exitpreis.
- [ ] **AC-3:** Vollständige Exits (FULL_EXIT) erscheinen weiterhin als roter Marker (ggf. visuell verbessert).
- [ ] **AC-4:** Stop-Anpassungen (STOP_TRAIL / Highwater-Mark-Update) erscheinen als horizontale gestrichelte Linie oder Marker auf der Stop-Preisebene, an der Zeitposition der Anpassung.
- [ ] **AC-5:** Jeder Marker hat einen Hover-Tooltip mit: Event-Typ, Zeitstempel, Preis, Stückzahl, PnL (wenn vorhanden), Exitgrund (wenn vorhanden).
- [ ] **AC-6:** Die Marker beeinflussen nicht die Chart-Skalierung (kein unkontrolliertes Y-Axis-Stretching durch weit entfernte Marker).
- [ ] **AC-7:** Bei 0 Trades erscheint kein Marker (kein Fehler, kein Crash).
- [ ] **AC-8:** Im Live-Modus werden Marker entsprechend dem aktuellen Trade-Status gezeigt (Entry vorhanden, Exit noch nicht = nur Entry-Marker sichtbar).
- [ ] **AC-9:** Alle neuen Marker-Typen haben `data-testid`-Attribute oder sind anderweitig für Playwright-Tests identifizierbar.

---

## Technische Details

### Bestandsaufnahme

**Existierende Klassen:**
- `TradingEventLayer` (`src/shared/components/IntraDayChart/layers/TradingEventLayer.ts`)
  - Aktuell: ENTRY (grün) + FULL_EXIT (rot) via `ISeriesMarkersPlugin` von lightweight-charts
  - Daten: `GET /api/v1/runs/{runId}/trades` → `TradeDto[]` → `ChartTradeEvent[]`
- `useChartData.ts`: lädt Bars, Indikatoren, Trades

**Daten-Analyse (vor Implementierung prüfen):**
- Was enthält `TradeDto`? Hat es Tranche-Information (T1/T2/T3)?
- Gibt es separate DTOs für Partial Exits und TRANCHE_ADD-Events?
- Gibt es Stop-Trail-Events im Event-Log (`eventType = "STOP_TRAILING"` o.ä.)?
- Falls Tranche-Daten im `/trades`-Endpoint fehlen: Endpoint erweitern ODER Event-Log-Endpoint (`/events?eventType=TRANCHE_ADD`) als zweite Quelle nutzen

### Marker-Typen und visuelle Spezifikation

| Event-Typ | Symbol | Farbe | Position |
|-----------|--------|-------|----------|
| ENTRY (T1, Erstpositionierung) | ↑ (Aufwärtspfeil, groß) | `#0EB35B` (grün) | Unterhalb der Bar |
| TRANCHE_ADD (T2/T3/...) | ◆ (Raute/Diamond) | `#22D36A` (hellgrün) | Unterhalb der Bar |
| PARTIAL_EXIT (Teilverkauf) | ↓ (Abwärtspfeil, mittel) | `#F59E0B` (amber/orange) | Oberhalb der Bar |
| FULL_EXIT (Vollausstieg) | ↓ (Abwärtspfeil, groß) | `#FC3243` (rot) | Oberhalb der Bar |
| STOP_TRAIL (Stop-Anpassung) | — (horizontale Linie, gestrichelt) | `#94A3B8` (slate/grau) | Auf Stop-Preislinie |

**Tooltip-Inhalt (bei Hover):**
```
TRANCHE_ADD T2
09:47 ET | $18.42 | 125 Shares
Avg. Entry: $18.35
```
```
PARTIAL_EXIT (T1 sold)
10:12 ET | $19.20 | 150 Shares
PnL: +$127.50 (+3.4%)
```
```
STOP_TRAIL
10:05 ET | Stop: $17.85 → $18.10
```

### Implementierungsansatz

**Option A — Daten aus `/trades`-Endpoint (bevorzugt wenn vollständig):**
- `TradeDto` um Tranche-Details erweitern: `tranches: TranchDto[]` (Entry-Zeit, Preis, Stück, PnL)
- `TradingEventLayer` iteriert alle Tranchen und erstellt Marker

**Option B — Daten aus Event-Log-Endpoint (falls `/trades` unvollständig):**
- Paralleler REST-Call: `GET /api/v1/backtests/{backtestId}/events?eventType=TRANCHE_ADD`
- Marker aus Event-Payloads konstruieren

**Für STOP_TRAIL-Ereignisse:**
- Diese sind höchstwahrscheinlich NICHT im `/trades`-Endpoint (Trades sind abgeschlossen, Stop-Anpassungen sind Zwischenereignisse)
- `GET /api/v1/backtests/{backtestId}/events?eventType=STOP_TRAILING` (oder ähnlicher Typ aus dem Event-Log)
- Stop-Trail-Marker als horizontale `PriceLine` in lightweight-charts oder als spezieller Marker

**lightweight-charts Marker-API:**
```typescript
// Marker-Typen aus lightweight-charts:
type SeriesMarkerShape = 'circle' | 'square' | 'arrowUp' | 'arrowDown';

// Marker-Objekt:
{
  time: UTCTimestamp,
  position: 'aboveBar' | 'belowBar' | 'inBar',
  color: string,
  shape: SeriesMarkerShape,
  text?: string,   // Kurztext auf dem Marker
  size?: number,   // Marker-Größe (1-4)
  id?: string,     // für Tooltip-Identifikation
}
```

**Tooltip-Implementierung:**
- Lightweight-charts v4 unterstützt custom tooltips über `chart.subscribeCrosshairMove()`
- Tooltip-Container: absolute-positioned div, erscheint near-crosshair bei hover über Marker

### Dateipfade (neue/geänderte Dateien)

- `src/shared/components/IntraDayChart/layers/TradingEventLayer.ts` — erweitern
- `src/shared/components/IntraDayChart/hooks/useChartData.ts` — ggf. zusätzliche API-Calls für Stop-Trail-Events
- `src/shared/components/IntraDayChart/components/TradeMarkerTooltip.tsx` — neu (Tooltip-Komponente)
- `src/shared/types/chart.ts` — neue Typen für `TrancheEvent`, `StopTrailEvent`
- Backend (falls nötig): `TradeDto` erweitern oder neuen Endpoint hinzufügen

---

## Konzept-Referenzen

| Dokument | Abschnitt |
|----------|-----------|
| `docs/concept/intraday-agent-concept.md` | Kap. 7: Strategy Logic (Multi-Tranche-Entry, Scaling-Out, Trailing Stop) |
| `docs/concept/intraday-agent-concept.md` | Kap. 15: Execution Policy (Partial Exits, Stop-Nachführung) |
| `docs/backend/architecture/07-oms.md` | Multi-Tranchen, Stop-Nachführung, Fill-Handling, P&L |
| `docs/frontend/architecture/09-frontend.md` | Chart-Komponente, Feature-Struktur |

---

## Guardrail-Referenzen

| Guardrail | Pfad |
|-----------|------|
| Frontend-Guardrail | `docs/frontend/guardrails/frontend.md` |
| CLAUDE.md | `T:/codebase/its_odin/CLAUDE.md` |

---

## Definition of Done

### 2.1 Code-Qualität
- [ ] Implementierung vollständig gemäß Akzeptanzkriterien
- [ ] Frontend: `npm run build` (Vite) fehlerfrei, TypeScript strict, keine `any`
- [ ] Backend (falls geändert): `mvn compile` fehlerfrei
- [ ] Keine Magic Numbers — Konstanten für Farben, Größen, Event-Typen
- [ ] JavaDoc auf neuen/geänderten Backend-Klassen (falls Backend geändert)
- [ ] Keine TODO/FIXME-Kommentare

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests für `TradingEventLayer` (gemockte TradeDto-Daten → erwartete Marker-Objekte)
- [ ] Unit-Tests für Tooltip-Rendering (korrekte Texte je Event-Typ)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Vitest-Komponenten-Test: `IntraDayChart` mit Mock-Trades zeigt die erwartete Anzahl Marker
- [ ] Backend (falls geändert): `*IntegrationTest` für erweiterten Trades-Endpoint

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT nach Edge-Cases gefragt: 0 Trades, 5 Tranchen, STOP_TRAIL bei fehlenden Event-Daten, Chart-Skalierung mit extremen Stop-Preisen
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini (drei Dimensionen)
- [ ] Dimension 1: Code-Review (Bugs, TypeScript-Typen, Rendering-Performance bei vielen Markern)
- [ ] Dimension 2: Konzepttreue (stimmen die dargestellten Events mit Kap. 7/15 überein?)
- [ ] Dimension 3: Praxis-Review (fehlen wichtige visuelle Elemente für den Nutzer?)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten erstellt und aktuell gehalten

### 2.8 Abschluss
- [ ] Commit mit aussagekräftiger Message
- [ ] Push auf Remote

---

## Notizen für den Implementierer

- **Zuerst Datenstruktur analysieren:** Bevor Marker-Code geschrieben wird, prüfen welche Felder `TradeDto` tatsächlich enthält (Tranche-Details vorhanden? Stop-Trail-Timestamps?). Ggf. das Backend-DTO und den Endpoint erweitern bevor das Frontend angepasst wird.
- **Stop-Trail ist ein Zwischen-Event:** Ein Stop-Trail-Ereignis ist kein Kauf/Verkauf, sondern eine Stop-Anpassung. Es liegt wahrscheinlich im Event-Log (eventType = "STOP_TRAIL" oder "STOP_ADJUSTED"), nicht im `/trades`-Endpoint. Den korrekten eventType-Wert im Code verifizieren.
- **lightweight-charts ISeriesMarkersPlugin:** Marker werden als Array gesetzt via `series.setMarkers(markers)`. Die Sortierung muss chronologisch (nach `time`) sein — sonst bricht lightweight-charts.
- **Chart-Skalierung:** Stop-Marker, die weit vom aktuellen Preis entfernt sind (z.B. Initial-Stop unter dem Tief), können den Chart vertikal strecken. Prüfen ob die Marker die Y-Achsen-Skalierung beeinflussen und ggf. `AutoscaleInfoProvider` anpassen.
- **Farben aus User-Vorgaben:** Bullish: `#0EB35B`, Bearish: `#FC3243` (aus MEMORY.md). Für TRANCHE_ADD: helleres Grün (damit unterscheidbar von Entry). Für PARTIAL_EXIT: Amber `#F59E0B`.
- **Tooltip-Bibliothek:** Keine externe Tooltip-Bibliothek einführen. Custom div mit absoluter Positionierung über `chart.subscribeCrosshairMove()` ist ausreichend.
- **Performance:** Bei Backtest-Läufen mit vielen Tranchen (z.B. 3 Instrumente × 5 Tranchen × mehrere Tage) können viele Marker entstehen. Sicherstellen dass `setMarkers()` nur einmal aufgerufen wird (nicht bei jedem Re-Render).
