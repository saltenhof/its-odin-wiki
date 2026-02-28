# ODIN-066: Backtesting Robustness (PBO, DSR, Monte Carlo)

## Modul

odin-backtest, odin-api

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

L

## Kontext

ODINs Regelwerk hat viele Freiheitsgrade (7 Gates, 4 Pattern-FSMs, 20 Subregimes, LLM-Tactical-Controls). Ohne formale Overfitting-Kontrolle ist die Wahrscheinlichkeit hoch, dass Backtest-Ergebnisse ueberschaetzt werden. Diese Story fuehrt drei akademische Standardmethoden ein: Deflated Sharpe Ratio (DSR) korrigiert fuer Multiple Testing, Monte Carlo Permutation Tests validieren statistische Signifikanz, und Neighbor-Tests pruefen Parameter-Stabilitaet. Damit werden Backtest-Ergebnisse erstmals statistisch belastbar.

## Scope

### In Scope

- **Deflated Sharpe Ratio (DSR):** Korrektur des beobachteten Sharpe Ratio fuer Anzahl getesteter Parameterkombinationen, Schiefe und Kurtosis der Returns
- **Monte Carlo Permutation Test:** Trade-Order-Shuffle (mindestens 1000 Permutationen) um p-Wert der beobachteten Sharpe/Returns zu berechnen
- **Neighbor-Test:** Parameter +/-10% Variation, Stabilitaetscheck ob Ergebnis robust bleibt
- Erweiterung von `BacktestReport`/`BacktestSummary` um Robustness-Metriken
- Konfigurierbare Anzahl Monte-Carlo-Iterationen und Neighbor-Schritte
- Ausgabe als Teil des bestehenden Backtest-Report-Formats

### Out of Scope

- Combinatorial Purged Cross-Validation (CPCV) — zu komplex fuer V1, erfordert Multi-Day-Partitioning
- Probability of Backtest Overfitting (PBO) im vollen Bailey-Sinne (benoetigt CPCV als Grundlage)
- Parameter-Grid-Search (automatische Optimierung) — nur Validation, nicht Optimierung
- UI-Darstellung der Robustness-Metriken (separate Story)
- Walk-Forward-Erweiterungen (bereits in ODIN-033 abgedeckt)

## Akzeptanzkriterien

- [ ] `DeflatedSharpeCalculator` berechnet DSR korrekt: `DSR = (observedSharpe - expectedMaxSharpe) / stdError` mit Korrektur fuer Schiefe/Kurtosis nach Bailey/Lopez de Prado (2014)
- [ ] `MonteCarloPermutationTest` fuehrt N (Default: 1000) Trade-Order-Shuffles durch und berechnet p-Wert: Anteil der Permutationen mit Sharpe >= beobachtetem Sharpe
- [ ] `NeighborStabilityTest` variiert konfigurierbare Parameter um +/-10% und berechnet Stabilitaets-Score: Anteil der Nachbar-Parametrisierungen mit positivem Sharpe
- [ ] `BacktestSummary` enthaelt neue Felder: `deflatedSharpe`, `monteCarloP`, `neighborStabilityScore`
- [ ] Unit-Test verifiziert: DSR fuer einen bekannten Return-Vektor (aus Bailey 2014 Paper) liefert korrekte Werte innerhalb Toleranz von 0.01
- [ ] Unit-Test verifiziert: Monte Carlo p-Wert fuer eine zufaellige Trade-Sequenz konvergiert gegen ~0.5 (+/-0.1)
- [ ] Unit-Test verifiziert: Neighbor-Stabilitaet fuer robuste Parameter nahe 1.0, fuer overfittete Parameter deutlich unter 0.5
- [ ] Monte-Carlo-Berechnung ist parallelisiert (ForkJoinPool oder parallel streams)
- [ ] Gesamte Robustness-Analyse laeuft als Post-Processing-Schritt nach dem regulaeren Backtest

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `DeflatedSharpeCalculator` | odin-backtest | `de.its.odin.backtest.robustness` | Berechnet DSR nach Bailey/Lopez de Prado. Input: observedSharpe, trialCount, returns[]. Output: deflatedSharpe (double) |
| `MonteCarloPermutationTest` | odin-backtest | `de.its.odin.backtest.robustness` | Shufflet Trade-Returns, berechnet p-Wert. Input: Trade-Returns[], numPermutations. Output: pValue (double) |
| `NeighborStabilityTest` | odin-backtest | `de.its.odin.backtest.robustness` | Variiert Parameter, fuehrt Re-Runs durch. Input: Baseline-Parameter, ParameterVariation-Definitionen. Output: stabilityScore (double) |
| `RobustnessReport` | odin-backtest | `de.its.odin.backtest.robustness` | Record: deflatedSharpe, monteCarloP, neighborStabilityScore, numPermutations, numNeighborVariants |
| `RobustnessRunner` | odin-backtest | `de.its.odin.backtest.robustness` | Orchestriert alle drei Tests. Wird nach BacktestRunner.run() aufgerufen |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `BacktestReport.BacktestSummary` (`de.its.odin.backtest.BacktestReport`) | Neue Felder: `Double deflatedSharpe`, `Double monteCarloP`, `Double neighborStabilityScore` (nullable, da optional) |
| `BacktestRunner` (`de.its.odin.backtest.BacktestRunner`) | Nach regulaerem Backtest-Lauf: `RobustnessRunner.analyze()` aufrufen wenn konfiguriert |
| `GovernanceVariant` (`de.its.odin.backtest.GovernanceVariant`) | Parameter-Extraktion fuer Neighbor-Test (Methode `extractVariableParameters()`) |

### Algorithmus: DSR (Bailey/Lopez de Prado 2014)

```
expectedMaxSharpe = sqrt(2 * ln(trialCount)) * (1 - gamma / (2 * ln(trialCount)))
    wobei gamma = Euler-Mascheroni-Konstante (0.5772...)
stdError = sqrt((1 - skew*SR/3 + kurtosis*SR^2/4) / n)
DSR = (observedSharpe - expectedMaxSharpe) / stdError
```

### Konfiguration

```properties
odin.backtest.robustness.enabled=false
odin.backtest.robustness.monte-carlo-iterations=1000
odin.backtest.robustness.neighbor-variation-percent=10
odin.backtest.robustness.min-trades-for-analysis=20
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 2: Backtesting-Robustheit (PBO, DSR, Monte Carlo), Abschnitte IST-Zustand, SOLL-Zustand, Quellen
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Berechnung (analog Indikator-Berechnungsarchitektur)

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen (neues Sub-Package `robustness` unter `de.its.odin.backtest`)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, Records fuer DTOs, Konstanten statt Magic Numbers

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-backtest`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.backtest.robustness.*`
- [ ] Port-Abstraktion eingehalten

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `DeflatedSharpeCalculator`, `MonteCarloPermutationTest`, `NeighborStabilityTest`
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer BacktestRunner (Neighbor-Test mockt die Re-Runs)
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Mindestens 1 Integrationstest der `RobustnessRunner` End-to-End mit synthetischen Trade-Daten durchspielt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen und Akzeptanzkriterien
- [ ] Edge Cases und Grenzfaelle diskutiert
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (mathematische Korrektheit, Numerik-Probleme)
- [ ] Dimension 2: Konzepttreue-Review (Bailey 2014 Paper vs. Implementierung)
- [ ] Dimension 3: Praxis-Review (realistische Trade-Mengen, Laufzeitverhalten)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Numerische Stabilitaet:** DSR-Berechnung verwendet Schiefe und Kurtosis der Return-Verteilung. Bei wenigen Trades (< 20) sind diese Schaetzer instabil — daher `minTradesForAnalysis` als Guard.
- **Monte Carlo Laufzeit:** 1000 Permutationen eines 100-Trade-Backtest sind O(100 * 1000) Shuffles — vernachlaessigbar. Aber: Neighbor-Test erfordert tatsaechliche Re-Runs des BacktestRunners, was teuer sein kann. Parallelisierung ist essentiell.
- **Neighbor-Test-Parameter:** Nicht alle Parameter sind sinnvoll zu variieren. Der Implementierer muss mit `GovernanceVariant` arbeiten, um die variablen Parameter zu identifizieren (z.B. Gate-Schwellen, ATR-Faktor, RSI-Levels).
- **Bailey 2014 Paper:** "The Deflated Sharpe Ratio: Correcting for Selection Bias, Backtest Overfitting, and Non-Normality" — die Formel ist klar definiert, aber die korrekte Schaetzung von `trialCount` (Anzahl aller jemals getesteten Parameterkombinationen) ist nicht-trivial. Als Naehrung: `trialCount = Anzahl der Backtest-Runs in der aktuellen Batch`.
- **Ergebnis-Interpretation:** DSR < 0 bedeutet: der beobachtete Sharpe ist statistisch nicht signifikant besser als Zufall nach Korrektur fuer Multiple Testing. Monte Carlo p > 0.05 bedeutet: Trade-Reihenfolge erklaert das Ergebnis moeglicherweise.
- **Diese Story ist L wegen Neighbor-Test:** DSR und Monte Carlo allein waeren M. Der Neighbor-Test erfordert Re-Runs und damit Integration in den BacktestRunner.
