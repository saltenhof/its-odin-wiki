# Protokoll: ODIN-117 — IntraDayChart backtest-live Modus mit Auto-Scroll und Day-Rollover

**Datum:** 2026-03-06
**Status:** Abgeschlossen (nach QA R1 Rework)

---

## Working State

- [x] Initiale Implementierung (R1)
- [x] Unit-Tests geschrieben (useAutoScroll: 13 Tests, dayRollover: 21 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] QA R1 Rework abgeschlossen (F-01 bis F-04)
- [x] Integrationstests fuer backtest-live Modus (QA F-03)
- [x] Commit & Push

---

## Erstellte / Geaenderte Dateien

### Neue Dateien (R1)

| Datei | Beschreibung |
|-------|-------------|
| `src/shared/components/IntraDayChart/hooks/useAutoScroll.ts` | Follow-Modus Hook: `scrollToRealTime()` nach jedem Bar, Freeze bei User-Scroll, Resume-Button |
| `src/shared/components/IntraDayChart/utils/dayRollover.ts` | Day-Rollover Utilities: `prepareDayRolloverData()`, `splitRolloverBars()`, `tagBars()` |
| `src/shared/components/IntraDayChart/hooks/useAutoScroll.test.ts` | 13 Unit-Tests fuer Auto-Scroll-State (initial state, scrollToLatest, resumeFollow, subscription, enabled-toggle) |
| `src/shared/components/IntraDayChart/utils/dayRollover.test.ts` | 21 Unit-Tests fuer Day-Rollover-Logik (normal case, edge cases, splitRolloverBars, tagBars) |

### Geaenderte Dateien (R1)

| Datei | Aenderung |
|-------|----------|
| `src/shared/components/IntraDayChart/types/chart.ts` | `ChartMode` erweitert um `'backtest-live'`; `BacktestChartEvent` Discriminated Union hinzugefuegt |
| `src/shared/components/IntraDayChart/IntraDayChart.tsx` | `backtest-live` Modus: useAutoScroll Integration, Day-Rollover Phase 1+2, Follow-Button, exchangeTimezone Prop |
| `src/shared/components/IntraDayChart/IntraDayChart.module.css` | `.followButton` Styles hinzugefuegt |
| `src/shared/components/IntraDayChart/IntraDayChart.integration.test.ts` | 6 Integrationstests fuer backtest-live Day-Rollover Pipeline hinzugefuegt (QA F-03) |

### Geaenderte Dateien (QA R1 Rework)

| Datei | Aenderung |
|-------|----------|
| `src/shared/components/IntraDayChart/hooks/useChart.ts` | `UseChartConfig.timezone` Feld hinzugefuegt; `buildChartOptions` setzt `timeScale.timezone` wenn vorhanden (QA F-02) |
| `src/shared/components/IntraDayChart/hooks/useChartData.ts` | `incrementEpoch()` Methode hinzugefuegt — inkrementiert chartEpoch ohne Datensatz-Reset (QA F-04) |
| `src/shared/components/IntraDayChart/panels/PricePanel.tsx` | `exchangeTimezone` Prop hinzugefuegt, in `useChart` Config geleitet (QA F-02) |
| `src/shared/components/IntraDayChart/panels/OscillatorPanel.tsx` | `exchangeTimezone` Prop hinzugefuegt (QA F-02) |
| `src/shared/components/IntraDayChart/panels/VolatilityPanel.tsx` | `exchangeTimezone` Prop hinzugefuegt (QA F-02) |
| `src/shared/components/IntraDayChart/panels/FlowPanel.tsx` | `exchangeTimezone` Prop hinzugefuegt (QA F-02) |
| `src/shared/components/IntraDayChart/IntraDayChart.tsx` | `incrementEpoch` aus useChartData destrukturiert, im Rollover Phase 1 Effect aufgerufen; `exchangeTimezone` an alle Panels weitergeleitet (QA F-02 + F-04) |

---

## Design-Entscheidungen

### D1: Zwei-Phasen Day-Rollover (pending → hydrated)

**Entscheidung:** Der Day-Rollover wird in zwei Effekt-Phasen aufgeteilt:
- **Phase 1:** `tradingDate`-Prop aendert sich → Vortags-Bars snapshotten, `pendingRolloverDateRef` setzen, leeren Kontext-Chart anzeigen.
- **Phase 2:** neue `data` fuer das pending Date eintrifft → `prepareDayRolloverData` mit neuem Tag aufrufen, `setRolloverData` setzen.

**Begruendung:** Eine einzelne Effekt-Ebene wuerde entweder `rolloverData` in die Dep-Array aufnehmen muessen (Infinite-Loop-Gefahr) oder die Datenkonsistenz verlieren. Die `pendingRolloverDateRef` als Ref (nicht State) verhindert den Dep-Loop: der zweite Effekt laueft nur bei `data`- oder `tradingDate`-Aenderungen.

### D2: `previousDayBarsRef` Snapshot-Timing

**Entscheidung:** Die Vortags-Bars werden in einem separaten Effekt kontinuierlich in `previousDayBarsRef` gespeichert. Phase 1 greift synchron auf diesen Ref zu, um den Snapshot zum richtigen Zeitpunkt zu erfassen.

**Begruendung:** Wenn Phase 1 auf den `data?.bars` State zugreifen wuerde (statt auf den Ref), koennte bereits die neue Tages-REST-Antwort eingetroffen sein. Der Ref haelt immer die *letzten* geladenen Bars des *aktuellen* (noch alten) Tages — der exakte Stand, den wir als Kontext wollen.

### D3: `prevBarCountRef.current = 0` beim Rollover

**Entscheidung:** Beim Day-Rollover Phase 1 wird `prevBarCountRef.current` auf 0 zurueckgesetzt.

**Begruendung (ChatGPT-Finding):** Ohne Reset wuerde Auto-Scroll erst aktiviert, wenn die neue-Tag-Bar-Zahl die Vortags-Bar-Zahl uebersteigt — also ein stundenlanger Dead-Scroll-Periode. Reset auf 0 stellt sicher, dass Follow-Mode vom ersten Bar des neuen Tages an aktiv ist.

### D4: `chartEpoch` Inkrementierung beim Rollover (QA F-04)

**Entscheidung:** `incrementEpoch()` wird in Phase 1 des Day-Rollovers aufgerufen (nach `setRolloverData`).

**Begruendung:** Ohne Epoch-Inkrementierung koennte `useChartLiveUpdates` noch Queued-Updates aus dem Vortag (epoch=N) verarbeiten und auf die neue Tages-Serie anwenden. Da `prepareDayRolloverData` effektiv `series.setData()` ausfuehrt, sind alle alten Queued-Updates semantisch invalide. Der Epoch-Bump verwirft sie zuverlassig.

### D5: `exchangeTimezone` an alle Panels (QA F-02)

**Entscheidung:** `exchangeTimezone` wird als Prop durch alle Panel-Komponenten (Price, Oscillator, Volatility, Flow) an `useChart` weitergegeben und dort als `timeScale.timezone` gesetzt.

**Begruendung:** Lightweight Charts zeigt die Timezone in der Time Scale (Crosshair-Label, Tick-Marks) pro Panel an. Wenn nur die PricePanel die Timezone erhaelt, zeigen Sub-Panels (Oscillator, Volatility) UTC oder eine andere Default-Timezone — inkonsistentes Bild. Alle vier Panels muessen dieselbe Exchange-Timezone erhalten.

### D6: `UseChartConfig.timezone` als optionales Feld

**Entscheidung:** `timezone` ist ein optionales Feld in `UseChartConfig`. Wenn nicht gesetzt, wird kein `timezone`-Property an `buildChartOptions` uebergeben.

**Begruendung:** Der `useChart` Hook wird auch in nicht-backtest-Kontexten verwendet (`history`, `live`). In diesen Faellen ist die Timezone irrelevant oder nicht bekannt. Das optionale Feld vermeidet Breaking Changes und erlaubt den bisherigen Default-Verhalten von Lightweight Charts.

---

## Offene Punkte

### A. `exchangeTimezone` in `live`-Modus noch nicht genutzt

Der `live`-Modus setzt `exchangeTimezone` derzeit auf `DEFAULT_EXCHANGE_TIMEZONE = 'America/New_York'` (Hardcode). Im backtest-live Modus wird der Wert aus der Prop gelesen. Eine saubere Loesung waere, die Timezone in beiden Modi aus der Prop zu lesen. Das ist Out-of-Scope fuer ODIN-117 (live-Mode-Verdrahtung ist ODIN-118+).

### B. Integrationstests testen Rendering-Schicht nicht

Da Lightweight Charts eine Canvas-API verwendet, die in jsdom nicht verfuegbar ist, testen die Integrationstests die Datenpipeline (prepareDayRolloverData → TradingEventLayer), nicht das tatsaechliche Chart-Rendering. Full-E2E-Tests (Playwright) sind ODIN-E2E vorbehalten.

---

## ChatGPT-Sparring

**Session:** 1x ChatGPT (owner=ODIN-117-worker), ts: 2026-03-06T06:32:16Z

**Thema:** Test-Edge-Cases fuer useAutoScroll und dayRollover

**Vorgeschlagene Edge-Cases und Bewertung:**

| Vorschlag | Umgesetzt? | Begruendung |
|-----------|-----------|-------------|
| `prevBarCountRef.current = 0` beim Rollover — sonst Dead-Scroll nach Tag-Wechsel | JA | Kritischer Bug ohne diesen Reset: Auto-Scroll schweigt bis neue-Tag-Bar-Anzahl > Vortags-Bar-Anzahl |
| Empty previous day (no context bars) Edge Case | JA | Test "empty prev day yields zero context bars" in Integration-Suite |
| Phase 1 / Phase 2 Race Condition bei schnellen tradingDate-Wechseln | DOKUMENTIERT | Guard: `pendingRolloverDateRef !== currentSimDateRef` in Phase 2 abfangen; Rapid-Skip wird korrekt behandelt |
| `useAutoScroll` bei `enabled=false` — kein Subscription | JA | Test "disabled=false: no subscription" in useAutoScroll.test.ts |
| Test "isFollowing true after scrollToLatest" | JA | Beweist korrekten State nach Follow-Resume |
| Infinite-Loop durch `rolloverData` in Dep-Array | VERMIEDEN | Design D1: `pendingRolloverDateRef` Ref statt State als Gate-Variable |
| `setData()` auf leerer Bars-Array crasht LWC | DOKUMENTIERT | Phase 1 zeigt Kontext-only Chart; `bars.length === 0` Guards in Panels |

---

## Gemini-Review

**Sessions:** 3x Gemini (owner=ODIN-117-worker), ts: 06:40:30, 06:43:44, 06:44:15Z

### Dimension 1: Code Bugs und Qualitaet

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | HIGH | `exchangeTimezone` wird nur als Display-Label verwendet, nicht in timeScale konfiguriert | QA R1: `timezone: exchangeTimezone` in `UseChartConfig` und `buildChartOptions` umgesetzt (F-02) |
| 2 | MEDIUM | TSDoc auf `BacktestChartEvent` unvollstaendig | EINGEARBEITET: TSDoc auf allen exportierten Typen ergaenzt |
| 3 | MEDIUM | `setRolloverData` ohne `chartEpoch`-Inkrementierung | QA R1: `incrementEpoch()` in Phase 1 Rollover hinzugefuegt (F-04) |
| 4 | LOW | Kein `isContext`-Flag in OhlcvBar-Serialisierung | BEWERTET: Flag ist Frontend-only, wird nie serialisiert; kein Problem |

### Dimension 2: Konzepttreue

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | HIGH | Story-Spec §Patterns: `chartEpoch wird inkrementiert` — fehlt in Implementierung | QA R1: `incrementEpoch()` implementiert und im Rollover-Effekt aufgerufen (F-04) |
| 2 | MEDIUM | `timeScale: { timezone: exchangeTimezone }` aus Story §Patterns fehlt | QA R1: Timezone-Config in useChart umgesetzt (F-02) |
| 3 | LOW | Kein Integrationstest fuer `backtest-live` Mode End-to-End | QA R1: 6 neue Integrationstests in IntraDayChart.integration.test.ts (F-03) |
| 4 | LOW | Protocol.md fehlt | QA R1: Diese Datei (F-01) |

### Dimension 3: Praxis-Gaps

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | HIGH | Day-Rollover ohne Epoch-Inkrementierung: stale Updates koennen auf neue Serie angewendet werden | QA R1: `incrementEpoch()` in Phase 1 (F-04) |
| 2 | HIGH | Timezone nur als Label — LWC zeigt UTC in Crosshair trotz `exchangeTimezone` Prop | QA R1: `timezone`-Config an LWC timeScale uebergeben (F-02) |
| 3 | MEDIUM | Phase 1 setzt `rolloverData` mit leeren Bars — kann LWC zu kurzem leerem Chart fuhren | DOKUMENTIERT: Bewusstes Design — leerer Chart fuer <100ms bis Phase 2 hydrated. Visuell akzeptabel. |
| 4 | LOW | Kein dedizierter Test fuer `backtest-live`-Modus in Integration-Suite | QA R1: 6 Tests hinzugefuegt (F-03) |

---

## Build-Status

```
npm run build: GREEN (245 Module, 0 Errors)
npm test --run: 512 Tests PASS, 2 pre-existierende Failures (useChartRecovery — ODIN-118 scope)
```

## Erstellte Commits

Siehe Git-Log des `its-odin-ui` und `its-odin-wiki` Repos.
