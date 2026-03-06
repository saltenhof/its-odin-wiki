# QA-Report ODIN-117 — Runde 1

**Ergebnis: FAIL**
**Datum: 2026-03-06**

## Prüfpunkte

### Build & Tests
- Build: GREEN (tsc + vite, 245 modules, 0 errors)
- Tests: 448 passed, 0 failed (28 test files)

### Hard Gates
- chatgpt_call: 1 (min 1) — OK
- gemini_call: 3 (min 3) — OK

### 5.1 Code-Qualität
- [x] TypeScript strict compliance — kein `any`, alle Interfaces mit `readonly`
- [x] Kein `enum`-Keyword — korrekt, Union Types verwendet (`ChartMode`, `BacktestChartEvent`)
- [x] CSS Modules — `IntraDayChart.module.css` korrekt erweitert
- [x] Keine TODO/FIXME-Kommentare
- [x] Code-Sprache Englisch, TSDoc auf allen exportierten Hooks/Interfaces/Funktionen
- [x] Discriminated Union für `BacktestChartEvent` (`type: 'day-rollover' | 'follow-change'`)
- [x] `IntraDayChartProps` als Discriminated Union korrekt erweitert um `backtest-live`-Variante

### 5.2 Unit-Tests (Vitest)
- [x] `useAutoScroll.test.ts` — 13 Tests (initial state, scrollToLatest, resumeFollow, subscription, enabled-toggle)
- [x] `dayRollover.test.ts` — 21 Tests (normal case, edge cases: empty prev/new day, custom contextBarCount, splitRolloverBars, tagBars)
- [x] Testkonventionen: `*.test.ts` (Vitest), kein Spring-Kontext

### 5.3 Integrationstests
- [x] `IntraDayChart.integration.test.ts` — 9 Tests (pre-existiert, wird nicht gebrochen)
- [ ] Kein dedizierter Integrationstest für `backtest-live` Mode (z.B. Day-Rollover-Sequenz mit IntraDayChart in `backtest-live` via renderHook)

### 5.4 DB-Tests
- Nicht zutreffend (reines Frontend)

### 5.5 ChatGPT-Sparring
- [x] 1 chatgpt_call in ODIN-117.jsonl (ts: 2026-03-06T06:32:16Z)
- Protokolldatei FEHLT — ChatGPT-Sparring-Abschnitt nicht dokumentiert (keine `protocol.md`)

### 5.6 Gemini-Review
- [x] 3 gemini_calls in ODIN-117.jsonl (ts: 06:40:30, 06:43:44, 06:44:15)
- Protokolldatei FEHLT — Gemini-Review-Abschnitt nicht dokumentiert (keine `protocol.md`)

### 5.7 Protokolldatei
- [ ] `protocol.md` fehlt vollständig im Verzeichnis `ODIN-117_intradaychart-backtest-live/`
- Nur `story.md` vorhanden — kein Working State, kein Design-Entscheidungen-Abschnitt, kein ChatGPT-/Gemini-Abschnitt

### 5.8 Abschluss
- [x] Commit `f290063` mit aussagekräftiger Message vorhanden
- [x] GitHub Issue #10 (ODIN-117) existiert mit vollständigem Issue-Body
- [x] `protocol.md` FEHLT → DoD nicht erfüllt

---

## Findings

| # | Schwere | Bereich | Finding | Erwartung |
|---|---------|---------|---------|-----------|
| F-01 | CRITICAL | DoD 5.7 + Abschluss | `protocol.md` fehlt vollständig. Das Wiki-Verzeichnis `ODIN-117_intradaychart-backtest-live/` enthält nur `story.md`. Working State, Design-Entscheidungen, ChatGPT-Sparring-Abschnitt und Gemini-Review-Abschnitt sind nicht dokumentiert. | `protocol.md` muss vollständig vorhanden sein (alle Pflichtabschnitte gemäß user-story-specification.md §3.2) |
| F-02 | MAJOR | DoD AC + 5.1 | `exchangeTimezone` Prop wird NICHT in die Lightweight Charts `timeScale`-Konfiguration übergeben. Der Wert wird nur als Anzeigelabel im Header verwendet (`{mode}{timezoneSuffix}`). Das Akzeptanzkriterium "Exchange-Timezone ist in der timeScale konfiguriert" und das Pattern `timeScale: { timezone: exchangeTimezone }` aus der Story sind damit nicht erfüllt. | `exchangeTimezone` muss als `timezone`-Parameter in der `createChart()`-Options oder via `chart.applyOptions({ timeScale: { timezone: exchangeTimezone } })` gesetzt werden |
| F-03 | MINOR | DoD 5.3 | Kein Integrationstest für den `backtest-live`-Modus im Zusammenspiel (Day-Rollover-Sequenz über `tradingDate`-Prop-Änderung, Follow-Button-Anzeige im Render). `IntraDayChart.integration.test.ts` testet den neuen Modus nicht. | Mindestens 1 Integrationstest der den `backtest-live`-Modus mit Day-Rollover End-to-End abdeckt |
| F-04 | MINOR | Konzepttreue | Story-Spec und Concept §10.4 erwähnen `chartEpoch`-Inkrementierung beim Day-Rollover. In der Implementierung wird `chartEpoch` beim Rollover nicht inkrementiert — `setRolloverData()` wird gesetzt, aber kein Epoch-Reset. Stale `useChartLiveUpdates`-Updates könnten nach Rollover nicht korrekt gefiltert werden. | `chartEpoch` sollte beim Day-Rollover inkrementiert werden, um queued Updates der vorherigen Tages-Serie zu verwerfen |

---

## Fazit

Die Implementierung ist technisch solide: Build GREEN, alle 448 Tests grün, Hard Gates erfüllt (1x ChatGPT, 3x Gemini), kein `any`, korrekte Union Types, gute Testabdeckung für die neuen Utilities. Das Hauptproblem ist **F-01 (CRITICAL)**: die `protocol.md` fehlt vollständig — ein nicht-verhandelbarer DoD-Pflichtpunkt. Zusätzlich ist **F-02 (MAJOR)** bedeutsam: `exchangeTimezone` wird nicht tatsächlich in die Chart-Engine übergeben, nur als Label angezeigt — das Akzeptanzkriterium ist damit unvollständig erfüllt. Story geht in Nacharbeit.
