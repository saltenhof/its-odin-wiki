# ODIN-070: LLM Confidence Calibration

## Modul

odin-brain, odin-api, odin-backtest

## Phase

1

## Abhaengigkeiten

Erfordert ausreichend Backtest-Daten mit vollstaendigem EventLog (>100 LLM-Analysen mit Outcome). Die Backtests muessen mit dem aktuellen LLM-Prompt und EventLog-Persistierung gelaufen sein.

## Geschaetzter Umfang

M

## Kontext

LLMs sind systematisch ueberoptimistisch kalibriert — ein `regime_confidence: 0.8` entspricht in der Realitaet oft nur 60-65% Korrektheit. ODINs Arbiter-Schwellen (0.5 fuer Regime-Nutzung, 0.7 fuer elevated Gate-Modus) basieren auf Design-Annahmen, nicht auf Empirie. Diese Story fuehrt isotonische Regression als Kalibrierungsfunktion ein und Brier Score als Monitoring-KPI, um die LLM-Confidence empirisch zu fundieren.

## Scope

### In Scope

- **Kalibrierungsfunktion via isotonische Regression:** Mapping von roher LLM-Confidence auf kalibrierte Wahrscheinlichkeit
- **Training auf historischen Backtest-Daten:** Pro Analyse-Aufruf: (reported_confidence, war_regime_korrekt_nach_N_Bars)
- **Brier Score als Monitoring-KPI:** `BS = mean((confidence - outcome)^2)` wobei outcome in {0, 1}
- **Expected Calibration Error (ECE)** als Zusatz-Metrik
- **Kalibrierungsfunktion persistieren und laden** (JSON-Serialisierung)
- **Integration in `DecisionArbiter`:** Kalibrierte Confidence statt roher Confidence verwenden

### Out of Scope

- Platt Scaling (logistische Regression) als Alternative — isotonische Regression ist besser fuer nicht-lineare Verzerrungen
- Live-Updates der Kalibrierung waehrend Trading (nur Offline-Training, vor RTH-Open laden)
- Reliability Diagrams als visuelle Darstellung (Frontend-Story)
- Kalibrierung von anderen LLM-Outputs als `confidenceScore` (z.B. `entryTiming`)

## Akzeptanzkriterien

- [ ] `ConfidenceCalibrator` implementiert isotonische Regression: Input (rawConfidence[]) -> kalibrierte Confidence
- [ ] `CalibrationTrainer` trainiert Kalibrierungsfunktion aus historischen Daten: Input-Paare (reported_confidence, actual_outcome) -> IsotonicRegressionModel
- [ ] `BrierScoreCalculator` berechnet Brier Score: `BS = (1/N) * sum((confidence_i - outcome_i)^2)`
- [ ] `EceCalculator` berechnet Expected Calibration Error: `ECE = sum(|bin_accuracy - bin_confidence| * bin_size / N)` mit 10 Bins
- [ ] `CalibrationDataExtractor` extrahiert Trainings-Paare aus EventLog: (LLM-Confidence, war Regime korrekt nach 3 Bars)
- [ ] `DecisionArbiter` verwendet kalibrierte Confidence wenn Kalibrierungsfunktion geladen
- [ ] Feature-Flag `odin.brain.calibration.enabled=false` (Default: aus)
- [ ] Kalibrierungsfunktion wird als JSON persistiert und beim Startup geladen
- [ ] Unit-Test verifiziert: Isotonische Regression fuer perfekt kalibrierte Daten liefert Identitaetsfunktion (calibrated ~= raw)
- [ ] Unit-Test verifiziert: Isotonische Regression fuer systematisch ueberoptimistische Daten (raw=0.8, actual=0.6) liefert calibrated < raw
- [ ] Unit-Test verifiziert: Brier Score fuer perfekte Vorhersagen = 0.0, fuer zufaellige = 0.25

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `ConfidenceCalibrator` | odin-brain | `de.its.odin.brain.calibration` | Wendet trainierte isotonische Regression an. Input: rawConfidence (double). Output: calibratedConfidence (double). Pro-Pipeline-POJO |
| `CalibrationTrainer` | odin-brain | `de.its.odin.brain.calibration` | Trainiert isotonische Regression. Input: List<CalibrationDataPoint>. Output: IsotonicRegressionModel |
| `IsotonicRegressionModel` | odin-brain | `de.its.odin.brain.calibration` | Record: sortierte Stuetzpunkte (x[], y[]) fuer monotone Interpolation. JSON-serialisierbar |
| `CalibrationDataPoint` | odin-brain | `de.its.odin.brain.calibration` | Record: rawConfidence (double), actualOutcome (boolean: war Regime korrekt?) |
| `BrierScoreCalculator` | odin-brain | `de.its.odin.brain.calibration` | Stateless Calculator: Brier Score aus Prediction-Outcome-Paaren |
| `EceCalculator` | odin-brain | `de.its.odin.brain.calibration` | Stateless Calculator: Expected Calibration Error mit konfigurierbarer Bin-Anzahl |
| `CalibrationDataExtractor` | odin-backtest | `de.its.odin.backtest.calibration` | Extrahiert CalibrationDataPoints aus EventLog (LLM-Analysen + 3-Bar-Forward-Outcomes) |
| `CalibrationStore` | odin-brain | `de.its.odin.brain.calibration` | Serialisiert/deserialisiert IsotonicRegressionModel als JSON |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `DecisionArbiter` (`de.its.odin.brain.arbiter.DecisionArbiter`) | Optionaler `ConfidenceCalibrator`. Wenn vorhanden: `calibratedConfidence = calibrator.calibrate(rawConfidence)` vor Schwellenvergleich |
| `LlmVote` (`de.its.odin.brain.arbiter.LlmVote`) | Neues optionales Feld `calibratedConfidence` (Double, nullable) fuer Audit-Trail |
| `BrainProperties` (`de.its.odin.brain.config.BrainProperties`) | Nested-Record `CalibrationProperties`: `enabled`, `modelPath`, `minDataPoints` (Default: 100) |
| `PipelineFactory` (`de.its.odin.core.pipeline.PipelineFactory`) | ConfidenceCalibrator erstellen und an DecisionArbiter uebergeben wenn konfiguriert |

### Isotonische Regression (Algorithmus)

Die isotonische Regression findet eine monoton steigende Funktion f(x) die die Summe der quadratischen Abweichungen minimiert. Implementierung: Pool Adjacent Violators Algorithm (PAVA).

```
1. Sortiere Datenpunkte nach rawConfidence aufsteigend
2. Initialisiere Bloecke: jeder Punkt ist ein eigener Block
3. Wiederhole bis keine Verletzungen:
   Wenn Block[i].mean > Block[i+1].mean:
       Merge Block[i] und Block[i+1] (gewichteter Mittelwert)
4. Ergebnis: monoton steigende Stuetzpunkte
```

### Konfiguration

```properties
odin.brain.calibration.enabled=false
odin.brain.calibration.model-path=data/calibration-model.json
odin.brain.calibration.min-data-points=100
odin.brain.calibration.ece-bins=10
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 7: LLM-Confidence-Kalibrierung, Abschnitte IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/05-llm-integration.md` — LLM-Output, Confidence-Felder
- `docs/backend/architecture/06-rules-engine.md` — Arbiter-Schwellenlogik, Confidence-Nutzung

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, Records fuer DTOs, Port-Abstraktion

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-api,odin-backtest`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.calibration.*`

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `ConfidenceCalibrator` (monotone Interpolation, Randwerte)
- [ ] Unit-Tests fuer `CalibrationTrainer` (PAVA-Algorithmus, Konvergenz)
- [ ] Unit-Tests fuer `BrierScoreCalculator` und `EceCalculator`
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Training -> Serialisierung -> Laden -> Kalibrierung End-to-End
- [ ] Integrationstest: DecisionArbiter mit kalibrierter Confidence
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen und Akzeptanzkriterien
- [ ] Edge Cases diskutiert (zu wenig Daten, alle Confidence-Werte gleich, extreme Werte)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Numerik, PAVA-Korrektheit)
- [ ] Dimension 2: Konzepttreue-Review (Brier Score Definition, Isotonic Regression)
- [ ] Dimension 3: Praxis-Review (Sample-Size-Anforderungen, Drift ueber Zeit)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **"War Regime korrekt nach 3 Bars":** Die Ground-Truth-Definition ist entscheidend. Vorschlag: LLM sagt TREND_UP. Nach 3 Bars: war der kumulative Return > 0? Dann "korrekt". Diese Heuristik ist bewusst simpel — komplexere Definitionen erfordern mehr Daten.
- **Mindestens 100 Datenpunkte:** Isotonische Regression mit weniger Punkten ist instabil. Das Feature-Flag bleibt deaktiviert bis genuegend Backtests gelaufen sind.
- **Kalibrierungs-Drift:** Die Kalibrierungsfunktion kann ueber Zeit driften (z.B. wenn das LLM-Modell aktualisiert wird). Monitoring via Brier Score: wenn BS signifikant steigt -> Rekalibrierung triggern.
- **Keine externe Library noetig:** PAVA (Pool Adjacent Violators Algorithm) ist ~30 Zeilen Code. Keine Abhaengigkeit auf ML-Libraries noetig.
- **Integration im Arbiter:** Die kalibrierte Confidence ersetzt die rohe NUR fuer die Schwellenentscheidung (0.5, 0.7). Die rohe Confidence wird weiterhin im EventLog gespeichert (Audit-Trail).
