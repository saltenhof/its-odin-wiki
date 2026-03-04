## Kontext

Nach ODIN-106 speichert die Datenbank ausschliesslich 1-Minuten-Bars. Die Chart-KPI-Berechnung (`IndicatorQueryService` + `IndicatorCalculationService`) erwartet jedoch 5-Minuten-Bars fuer ta4j-basierte Indikatoren (EMA50/100, RSI, ATR, Bollinger, ADX) und liefert leere Ergebnisse wenn keine 5-Minuten-Bars vorhanden sind. Dieses Problem trat konkret bei IREN 2026-02-24 auf: Der Backtest lud nur 1-Minuten-Bars, danach fehlten alle Chart-KPIs. Diese Story schliesst die Luecke, indem der `IndicatorQueryService` die benoetigten 5-Minuten-Bars on-the-fly aus 1-Minuten-Bars aggregiert — mittels der bereits existierenden `BarAggregationService.aggregateOneMinToFiveMin()`.

## Modul
odin-app, odin-brain

## Abhaengigkeiten
ODIN-106 (Unified 1-Min-Only Data Pipeline)

## Geschaetzter Umfang
S

---

## Scope

### In Scope
- **IndicatorQueryService:** `calculateOnTheFly()` laedt nur noch 1-Minuten-Bars aus DB und aggregiert 5-Minuten-Bars on-the-fly via `BarAggregationService`
- **IndicatorCalculationService:** Guard-Clause anpassen — wenn 5-Minuten-Bars leer aber 1-Minuten-Bars vorhanden, soll trotzdem berechnet werden (VWAP aus 1-min, ta4j-Indikatoren aus aggregierten 5-min)
- **ChartController/ChartDataService:** Falls 5-Minuten-Bars direkt aus DB geladen werden, auf Aggregation umstellen

### Out of Scope
- Aenderungen an der Download-Pipeline (ODIN-106)
- Indicator Write Path / Persistierung (ODIN-108)
- Neue Aggregations-Methoden (1m→3m existiert noch nicht, wird hier nicht benoetigt)
- Aenderungen an BarAggregationService (genuegt in aktueller Form)

## Akzeptanzkriterien

- [ ] `IndicatorQueryService.calculateOnTheFly()` laedt nur 1-Minuten-Bars aus DB
- [ ] 5-Minuten-Bars werden on-the-fly via `BarAggregationService.aggregateOneMinToFiveMin()` erzeugt
- [ ] Chart-KPIs (VWAP, EMA, RSI, ATR, Bollinger, ADX) werden korrekt berechnet auch wenn nur 1-Minuten-Bars in der DB liegen
- [ ] Die KPI-Werte sind numerisch identisch zu den bisherigen Werten (gleiche Aggregationslogik: O=first, H=max, L=min, C=last, V=sum)
- [ ] VWAP wird weiterhin aus 1-Minuten-Bars berechnet (hoechste Aufloesung, smooth curve)
- [ ] Keine Regression: Bestehende Chart-Darstellung fuer Tage MIT 5-Minuten-Bars in DB funktioniert weiterhin (Uebergangsphase)
- [ ] Unit-Test: calculateOnTheFly mit nur 1-min Bars liefert vollstaendige Indikatoren
- [ ] Integrationstest: Chart-API fuer IREN 2026-02-24 liefert KPIs (aktuell fehlerhaft)

## Technische Details

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Aenderung |
|--------|-------|-----------|
| `IndicatorQueryService` | odin-app | `calculateOnTheFly()`: Statt `loadBars(symbol, date, 300)` fuer 5-min → `loadBars(symbol, date, 60)` fuer 1-min, dann `barAggregationService.aggregateOneMinToFiveMin(oneMinBars)` aufrufen. Neue Dependency auf `BarAggregationService`. |
| `IndicatorCalculationService` | odin-brain | `calculateFromBars()` Zeile 94: Guard-Clause `fiveMinBars.isEmpty() → return emptyList()` anpassen — wenn 5-min leer aber 1-min vorhanden, VWAP trotzdem berechenbar. `calculateAtFiveMinWithHighResVwap()` Zeile 172: gleiche Anpassung. |

### Neue Klassen

Keine

### Konfiguration

Keine

## Konzept-Referenzen
- ODIN-106 — Voraussetzung: Nur 1-Minuten-Bars in DB
- `BarAggregationService` (`odin-app/.../service/BarAggregationService.java`) — Bestehende Aggregationslogik: `aggregateOneMinToFiveMin()` gruppiert je 5 Bars, OHLCV-Standard-Aggregation
- `IndicatorCalculationService` (`odin-brain/.../kpi/IndicatorCalculationService.java`) — `calculateAtFiveMinWithHighResVwap()` ist der primaere Berechnungspfad fuer 5-min Charts

## Guardrail-Referenzen
- `CLAUDE.md` — Backend Coding Rules: Kein `var`, JavaDoc, Records
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

## Notizen fuer den Implementierer

1. **BarAggregationService ist Spring-Bean** (in odin-app). `IndicatorQueryService` kann sie direkt per Constructor-Injection nutzen — beide sind im selben Modul.
2. **IndicatorCalculationService ist in odin-brain** und kein Spring-Bean (wird manuell instanziiert). Aenderungen dort beschraenken sich auf Guard-Clause-Anpassungen; die eigentliche Aggregation findet in odin-app statt.
3. **Numerische Konsistenz pruefen:** Die aggregierten 5-min-Bars muessen identisch sein zu den frueher von Yahoo direkt geladenen 5-min-Bars. Testfall: Tag mit beiden Datenbestaenden (z.B. IREN 2026-02-23) vergleichen.
4. **loadPersistedIndicators()** ruft ebenfalls `calculateOnTheFly()` als Fallback und fuer Warmup-Fill auf — profitiert automatisch von der Aenderung.

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Test fuer `IndicatorQueryService.calculateOnTheFly()` mit nur 1-min Bars
- [ ] Unit-Test fuer `IndicatorCalculationService` mit aggregierten 5-min Bars
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Chart-API Endpoint liefert KPIs fuer Tag mit nur 1-min Bars
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet
- [ ] Edge Cases identifiziert und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch ChatGPT — Drei Dimensionen
- [ ] Dimension 1: Code-Review
- [ ] Dimension 2: Konzepttreue-Review
- [ ] Dimension 3: Praxis-Review
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`
