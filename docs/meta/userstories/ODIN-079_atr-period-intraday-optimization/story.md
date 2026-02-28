# ODIN-079: ATR Period Intraday Optimization

**Modul:** odin-brain
**Phase:** 1
**Abhaengigkeiten:** Keine
**Geschaetzter Umfang:** S

---

## Kontext

ODIN verwendet ATR(14), ADX(14) und RSI(14) auf 5-Minuten-Bars — Standard-Perioden die fuer Daily-Charts konzipiert wurden. Auf 5m-Bars entspricht ATR(14) einem 70-Minuten-Lookback, was moeglicherweise zu traege fuer Intraday-Regimewechsel ist. ATR(10) = 50 Min, ATR(7) = 35 Min waeren responsiver. Die Research-Literatur empfiehlt kein bestimmtes Optimum, sondern einen systematischen Backtest-Vergleich. Diese Story macht die Perioden konfigurierbar und dokumentiert die Backtest-Methodik.

## Scope

**In Scope:**

- Sicherstellen dass ATR-Periode, ADX-Periode und RSI-Periode vollstaendig konfigurierbar sind (ueber `BrainProperties.KpiProperties`)
- Verifizieren dass bestehende Properties (`atrPeriod`, `adxPeriod`, `rsiPeriod`) korrekt durch die KPI-Berechnungskette propagiert werden
- Dokumentation einer Backtest-Vergleichsmethodik (ATR(14) vs. ATR(10) vs. ATR(7)) im `protocol.md`
- Sicherstellen dass Perioden-Aenderung zur Laufzeit ohne Code-Aenderung moeglich ist (nur Properties)
- Unit-Tests die verifizieren dass verschiedene Perioden zu unterschiedlichen KPI-Werten fuehren

**Out of Scope:**

- Durchfuehrung der eigentlichen Backtest-Vergleiche (erfordert ausreichend historische Daten)
- Automatische Perioden-Selektion basierend auf Volatilitaet
- Multi-Timeframe-ATR (verschiedene Perioden pro Timeframe) — aktuell wird eine Periode fuer alle Timeframes verwendet
- Aenderung der Default-Werte (bleiben auf 14 bis Backtest-Ergebnisse vorliegen)

## Akzeptanzkriterien

- [ ] `BrainProperties.KpiProperties` enthaelt bereits `atrPeriod`, `rsiPeriod`, `adxPeriod` — verifizieren dass diese korrekt durch `KpiEngine` bis in die ta4j-Indikator-Konstruktoren propagiert werden
- [ ] Unit-Test: KpiEngine mit `atrPeriod=10` liefert andere ATR-Werte als mit `atrPeriod=14` fuer dieselben Bars
- [ ] Unit-Test: KpiEngine mit `rsiPeriod=9` liefert andere RSI-Werte als mit `rsiPeriod=14`
- [ ] Unit-Test: KpiEngine mit `adxPeriod=10` liefert andere ADX-Werte als mit `adxPeriod=14`
- [ ] Verifizieren: Alle Stellen die auf ATR/RSI/ADX-Werte zugreifen, verwenden die Werte aus `IndicatorResult` (keine hardcodierten Perioden an anderer Stelle)
- [ ] Kein Verwendungsort im Code referenziert direkt die Periode (z.B. keine Stelle die `atr14_5m` als String mit eingebetteter "14" erstellt und dann mit einer anderen Periode berechnet)
- [ ] Dokumentation im `protocol.md`: Backtest-Vergleichsmethodik — welche Metriken (Sharpe, Win-Rate, Max-DD, Avg-Slippage), welche Parameter-Kombinationen, Neighbor-Test +-10%

## Technische Details

### Bestehende Klassen (werden geprueft/erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `BrainProperties.KpiProperties` | `odin-brain/.../config/BrainProperties.java` | Bereits vorhanden: `atrPeriod`, `rsiPeriod`, `adxPeriod` — Verifizierung der Propagation |
| `KpiEngine` | `odin-brain/.../kpi/KpiEngine.java` | Verifizierung: ta4j-Indikator-Konstruktoren nutzen konfigurierte Perioden |
| `IndicatorResult` | `odin-api/.../dto/IndicatorResult.java` | Pruefung: Feldnamen wie `rsi14_5m` sind semantisch korrekt (Feld traegt die konfigurierte Periode, nicht zwingend "14") |

### Keine neuen Klassen

Diese Story ist primaer eine Verifikations- und Dokumentations-Story. Falls bei der Verifikation Luecken gefunden werden (z.B. hardcodierte Perioden), werden diese behoben.

### Potenzielle Findings

1. **Feld-Naming in `IndicatorResult`:** Felder heissen `rsi14_5m`, `atr14_5m`, `adx14_5m` — die "14" ist im Feldnamen eingebacken. Wenn die Periode auf 10 geaendert wird, ist der Feldname irrefuehrend. **Entscheidung des Implementierers:** Entweder Felder umbenennen (z.B. `rsi_5m`, `atr_5m`) oder einen Kommentar hinzufuegen dass der Feldname die Default-Periode referenziert aber der tatsaechliche Wert konfigurierbar ist.
2. **RegimeResolver-Schwellen:** `BrainProperties.RulesProperties.RegimeProperties` enthaelt `adxTrendThreshold` etc. — diese muessen unabhaengig von der ADX-Periode sinnvoll sein. Eine kuerzere ADX-Periode liefert volatilere Werte → die Schwellen muessen ggf. angepasst werden.

### Konfiguration

Bereits vorhanden in `application-brain.properties`:
```properties
odin.brain.kpi.atr-period=14
odin.brain.kpi.rsi-period=14
odin.brain.kpi.adx-period=14
```

## Konzept-Referenzen

- `theme-backlog.md` Thema 17: "ATR-Periode Intraday-Optimierung (14 vs. 10 vs. 7)" — IST-Zustand, SOLL-Zustand, Backtest-Pflicht
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Engine, ta4j-Integration, Indikator-Berechnung
- Research-Quant Kap. 7.2 und Kap. 1.1: Hoehere Responsivitaet != besser, daher Backtest-Pflicht

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints
- `CLAUDE.md` — Coding-Regeln (keine Magic Numbers, konfigurierbare Parameter via Properties)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Verifikation abgeschlossen, eventuelle Luecken behoben
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — Perioden ausschliesslich aus Properties
- [ ] JavaDoc aktualisiert wo noetig
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer verschiedene Perioden-Konfigurationen in KpiEngine
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: KpiEngine mit nicht-default Perioden liefert valide IndicatorResults
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session: Backtest-Methodik fuer Perioden-Vergleich besprechen
- [ ] Edge Cases fuer kurze Perioden (ATR(7) warmup, RSI(9) Rauschen) abfragen
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (propagierte Perioden, ta4j-Korrektheit)
- [ ] Dimension 2: Konzepttreue-Review (Vergleich mit Thema 17)
- [ ] Dimension 3: Praxis-Review (Regime-Schwellen bei geaenderten Perioden)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit Verifikationsergebnis, Backtest-Methodik-Dokumentation, Findings

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Primaer eine Verifikations-Story:** Der Implementierer soll den Datenpfad von `BrainProperties.KpiProperties.atrPeriod` bis zum finalen `IndicatorResult.atr14_5m`-Wert nachverfolgen und sicherstellen, dass an keiner Stelle eine "14" hardcodiert ist.
- **Feldnamen-Entscheidung:** Die Felder in `IndicatorResult` heissen aktuell `atr14_5m`, `rsi14_5m`, `adx14_5m` etc. Das Umbenennen ist ein Breaking Change fuer alle Konsumenten (RulesEngine, RegimeResolver, GateCascadeEvaluator, PromptBuilder, IndicatorSnapshotEntity, Frontend-DTOs). Empfehlung: Nicht umbenennen in dieser Story, aber einen JavaDoc-Kommentar ergaenzen dass "14" der Default ist und die tatsaechliche Periode konfigurierbar.
- **Backtest-Methodik im protocol.md:** Kein Code — nur Dokumentation. Ziel ist, dass der naechste Agent, der den Backtest durchfuehrt, genau weiss was zu tun ist: welche Parameter-Kombinationen, welche Metriken, welche Schwellen fuer "besser/schlechter".
- **Vorsicht bei Schwellen-Wechselwirkung:** Wenn ATR-Periode von 14 auf 10 geaendert wird, aendern sich ATR-Werte. Alle Schwellen die in ATR-Vielfachen ausgedrueckt sind (z.B. `stopLossAtrFactor`, `trailingStopAtrFactor`, `vwapStandardAtrFactor`) sollten robuster sein (Verhaeltnis bleibt aehnlich). Schwellen die in absoluten Werten ausgedrueckt sind, koennten betroffen sein.
