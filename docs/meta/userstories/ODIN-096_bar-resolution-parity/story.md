# ODIN-096: Bar-Resolution-Paritaet — Einheitlicher Datenpfad Live und Backtest

> **GitHub Issue:** saltenhof/its-odin-backend#97
> **Story-ID:** ODIN-096
> **Status:** Todo
> **Size:** L
> **Module:** odin-data, odin-brain, odin-core, odin-backtest
> **Epic:** Data Pipeline
> **Erstellt:** 2026-03-02

---

## Kontext

Der Backtest nutzt einen komplett anderen Datenpfad als der Live-Modus: Live-Bars durchlaufen BarBuilder → BarAggregator → RollingDataBuffer, waehrend Backtest-Bars via BAR_APPROXIMATED direkt in den 5m-Buffer geschrieben werden. Kritische Komponenten (S/R Engine, VPIN, Monitor Events, Opening Patterns) sind im Backtest vollstaendig inaktiv, kurzfristige Indikatoren (EMA9/21, VolumeRatio) arbeiten auf 5m statt 3m. Ein Backtest-Ergebnis, das auf diesen Luecken basiert, kann keine valide Aussage ueber das Live-Verhalten treffen. Diese Story stellt die Datenpfad-Paritaet her, damit der Backtest als zuverlaessiger Praediktor fuer Live-Verhalten dienen kann.

## Modul

odin-data, odin-brain, odin-core, odin-backtest

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

L

---

## Scope

### In Scope

- `BarInterval`-Enum um `THREE_MINUTES(180)` erweitern
- Backtest-Default auf 1-Minuten-Bars umstellen (`interval_seconds=60` aus `odin.market_bar`)
- `BAR_APPROXIMATED`-Bypass in `DataPipelineService` entfernen — alle Bars (Live und Backtest) durch denselben `BarAggregator`-Pfad leiten
- `RollingDataBuffer.getDecisionBars()` dynamisch anhand des konfigurierten `decision-bar-timeframe-s` aufloesen (statt hardcoded 3m-Buffer)
- `BarSeriesManager.resolveDuration()` konfigurierbar statt hardcoded 180s
- `SimulationRunner` SimClock-Stepping auf 60s (1-Minuten-Intervall)
- Backtest-Datenpfad laedt 1m-Bars aus DB; Fail-Fast mit klarer Fehlermeldung wenn keine 1m-Bars vorhanden
- `decision-bar-timeframe-s` als einheitlicher Default fuer Live und Backtest in Properties
- `BAR_APPROXIMATED`-Flag behaelt semantische Bedeutung (kein Tick-Level vorhanden), steuert aber nicht mehr den Datenpfad

### Out of Scope

- Aenderung des `decision-bar-timeframe-s` Wertes (bleibt bei 180s / 3m)
- Entfernung von 5m-Bars (bleiben fuer RSI, ATR, ADX, Bollinger, EMA50/100)
- OFI-Berechnung im Backtest (erfordert Tick-Level-Daten, nicht nur OHLCV)
- Download/Beschaffung von 1m-Bars (separate Concern, provider-agnostisch ueber HistoricalDataProvider)
- CachedAnalyst-Optimierung
- 15m/30m-Bars explizit (werden automatisch vom BarAggregator produziert)

## Akzeptanzkriterien

- [ ] Backtest laedt 1-Minuten-Bars (`interval_seconds=60`) aus `odin.market_bar`
- [ ] `snapshot.oneMinBars()` ist im Backtest NICHT leer — S/R Engine, VPIN, Opening Pattern Detector erhalten Daten
- [ ] `buffer.getThreeMinBarCount() > 0` im Backtest — 3m-Bars werden vom BarAggregator produziert
- [ ] `getDecisionBars()` liefert Bars gemaess konfiguriertem `decision-bar-timeframe-s` (nicht hardcoded 3m)
- [ ] KpiEngine nutzt `seriesDecision` statt `series5m`-Fallback fuer EMA9/21, VolumeRatio
- [ ] Monitor Events werden im Backtest detektiert (MonitorEvent-Logeintraege vorhanden)
- [ ] VPIN wird im Backtest berechnet (`snapshot.vpin() != NaN`)
- [ ] LLM-Sampling-Frequenz konsistent: bei `intervalBars=5` und 3m-Decision-Bars ca. 15 Min Intervall
- [ ] SimClock stepped in 60s-Schritten (nicht 300s)
- [ ] Backtest schlaegt mit klarer Fehlermeldung fehl wenn keine 1m-Bars fuer Symbol/Datum vorhanden: `"No 1-minute bars available for {symbol} on {date}. Download required."`
- [ ] Kein stiller Fallback auf 5m-Bars
- [ ] `BAR_APPROXIMATED`-Flag steuert NICHT mehr den Datenpfad, nur noch semantische Markierung (z.B. OFI-Skip)
- [ ] Live-Modus ist NICHT beeintraechtigt — alle bestehenden Live-Tests bestehen weiterhin

## Technische Details

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Package | Aenderung |
|--------|-------|---------|-----------|
| `BarInterval` | odin-backtest | `de.its.odin.backtest` | Enum-Wert `THREE_MINUTES(180)` hinzufuegen |
| `DataPipelineService` | odin-data | `de.its.odin.data.service` | Zeilen 603-605: `BAR_APPROXIMATED`-Bypass entfernen. Alle Bars (auch approximated) durch `BarAggregator.onOneMinBar()` leiten. Flag nur noch als semantischer Hinweis (z.B. OFI-Skip) |
| `RollingDataBuffer` | odin-data | `de.its.odin.data.buffer` | `getDecisionBars()` (Zeile 159): Statt hardcoded `threeMinBars.toList()` dynamisch anhand `decision-bar-timeframe-s` den korrekten Buffer zurueckgeben. Konfigurationswert injizieren |
| `BarAggregator` | odin-data | `de.its.odin.data.buffer` | Verifizieren, dass `onOneMinBar()` korrekt 3m/5m/15m/30m aggregiert (bereits implementiert fuer Live) |
| `BarSeriesManager` | odin-brain | `de.its.odin.brain.kpi` | `resolveDuration()` Zeilen 157-164: `TIMEFRAME_DECISION` statt hardcoded `DURATION_3M_SECONDS` aus Konfiguration (`decision-bar-timeframe-s`) lesen |
| `SimulationRunner` | odin-core | `de.its.odin.core.sim` | `barIntervalSeconds` von 300 auf 60 aendern. SimClock stepped in 1m-Schritten |
| Backtest-Datenpfad | odin-backtest / odin-data | — | Bars mit `interval_seconds=60` aus `odin.market_bar` laden statt `interval_seconds=300`. Fail-Fast wenn keine 1m-Bars vorhanden |

### Konfiguration

```properties
# Bereits vorhanden — wird jetzt einheitlich fuer Live UND Backtest genutzt:
odin.data.pipeline.decision-bar-timeframe-s=180

# Backtest-Default aendern:
odin.backtest.bar-interval-seconds=60
```

### Architektur-Diagramm (Zielzustand)

```
Beide Modi:  1m-Bars → BarAggregator(3m/5m/15m/30m) → RollingDataBuffer → Snapshot
```

Der BarAggregator ist der EINZIGE Pfad fuer Bar-Aggregation — kein Bypass, kein Sonderweg.

## Konzept-Referenzen

- `docs/concept/adaptive-llm-sampling/bar-resolution-parity.md` — Abschnitt 1: Problemstellung (1.1 Designprinzip, 1.2 Ist-Zustand mit Diskrepanz-Tabelle, 1.3 Code-Stellen der Diskrepanz)
- `docs/concept/adaptive-llm-sampling/bar-resolution-parity.md` — Abschnitt 2: Loesung (2.1-2.5 alle fuenf Anforderungen)
- `docs/concept/adaptive-llm-sampling/bar-resolution-parity.md` — Abschnitt 3: Aenderungskatalog (3.1 Anforderungstabelle, 3.2 Validierungskriterien, 3.4 Nicht-Aenderungen)
- `docs/concept/adaptive-llm-sampling/bar-resolution-parity.md` — Abschnitt 4: Laufzeit-Impact (4.1 Vergleichstabelle 5m vs 1m)

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln zwischen Modulen
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln R1-R13, CSpec (Konfigurationsnamespace), Test-Konventionen
- `CLAUDE.md` — "Use MarketClock — no Instant.now() in trading code paths"

## Notizen fuer den Implementierer

1. **BAR_APPROXIMATED ist der Dreh- und Angelpunkt.** Der Bypass in `DataPipelineService` (Zeilen 603-605) ist die Hauptursache. Entferne den Sonderpfad, leite alle Bars durch `BarAggregator.onOneMinBar()`. Das Flag bleibt als semantischer Marker bestehen (OFI-Berechnung ueberspringen, da keine Tick-Daten).

2. **getDecisionBars() Injection:** `RollingDataBuffer` ist vermutlich kein Spring Bean. Pruefe, wie der konfigurierte Timeframe dorthin gelangt — entweder via Constructor-Injection beim Buffer-Erzeugen oder als Konfigurationsobjekt. Keine statischen Felder.

3. **Rueckwaertskompatibilitaet Live:** Der Live-Modus darf NICHT brechen. Alle bestehenden Live-Tests muessen weiterhin bestehen. Die Aenderungen betreffen primaer den Backtest-Pfad; der Live-Pfad sollte bereits korrekt durch BarAggregator laufen.

4. **SimulationRunner Timing:** Mit 390 statt 78 Iterationen pro Handelstag steigt die reine Sim-Logik-Zeit auf ca. 0.5s — vernachlaessigbar. Der limitierende Faktor sind LLM-Calls (ca. 26 statt 16 bei 3m Decision-Bars).

5. **Datenvoraussetzung:** 1-Minuten-Bars muessen in `odin.market_bar` mit `interval_seconds=60` vorliegen. Falls fuer Test-Tage keine 1m-Daten existieren, muessen diese VOR dem Backtest-Test heruntergeladen werden (nicht Scope dieser Story, aber Voraussetzung fuer Integrationstests).

6. **A/B-Vergleich:** Nach Implementierung einen Backtest-Vergleichslauf durchfuehren — gleicher Tag, 1m (neu) vs. 5m (alt). Abweichungen dokumentieren und erklaeren. Alte Ergebnisse als "v1 (5m-only)" kennzeichnen.

7. **Hardcoded Stellen:** Konzept listet 4 konkrete Code-Stellen (Abschnitt 1.3). Alle vier muessen adressiert werden — keine uebersehen.

## Definition of Done

### Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer alle geaenderten Klassen mit Geschaeftslogik
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT
- [ ] Spezifisch: `RollingDataBuffer.getDecisionBars()` mit verschiedenen Timeframe-Konfigurationen testen
- [ ] Spezifisch: `DataPipelineService` ohne BAR_APPROXIMATED-Bypass testen (1m-Bars gehen durch BarAggregator)
- [ ] Spezifisch: `BarSeriesManager.resolveDuration()` mit konfiguriertem Wert testen

### Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstests mit realen Klassen
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest: vollstaendiger Backtest-Durchlauf mit 1m-Bars der alle Validierungskriterien prueft

### Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: geaenderte Klassen, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Implementierungsfehler, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (Implementierung vs. Konzept bar-resolution-parity.md)
- [ ] Dimension 3: Praxis-Review (unbehandelte reale Szenarien)
- [ ] Findings bewertet und berechtigte Findings behoben

### Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten (Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review)

### Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`
