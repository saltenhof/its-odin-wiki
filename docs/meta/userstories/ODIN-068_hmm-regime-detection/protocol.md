# Protokoll: ODIN-068 -- HMM Regime Detection

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### Eigene HMM-Implementierung statt Library
Apache Commons Math 3 hat kein HMM-Modul. Smile ML hat eines, aber es wuerde eine grosse neue Dependency einbringen. Da Forward-Algorithmus und Baum-Welch jeweils ~50 Zeilen sind (wie in der Story angegeben), implementieren wir selbst. Dies haelt die Dependency-Oberflaeche sauber.

### Log-Space-Berechnung
Der Forward-Algorithmus wird im Log-Space implementiert (Log-Sum-Exp Trick), da die Multiplikation vieler kleiner Wahrscheinlichkeiten nach wenigen hundert Schritten zu numerischem Unterlauf fuehrt (Rabiner 1989, Section III-A).

### RegimeProbabilities in odin-api
Das Record `RegimeProbabilities` wird in `de.its.odin.api.dto` platziert (wie von der Story spezifiziert), da es moduluebergreifend als DTO benoetigt wird.

### IndicatorResult-Erweiterung
`IndicatorResult` erhaelt ein optionales Feld `RegimeProbabilities hmmRegime` fuer Downstream-Logging. Default ist `null`.

### Integration in RegimeResolver
Der RegimeResolver erhaelt einen optionalen `HmmRegimeDetector`. Wenn vorhanden und HMM aktiviert, wird das HMM-Regime anstelle des deterministischen Regimes verwendet. Die Subregime-Bestimmung bleibt deterministisch.

### HmmParameterStore: File-basiert
Parameter werden als JSON in eine konfigurierbare Datei serialisiert. Jackson ObjectMapper wird fuer Serialisierung verwendet (bereits als transitive Spring-Dependency verfuegbar).

### PipelineFactory-Integration
Die `PipelineFactory` erstellt beim Pipeline-Aufbau optional einen `HmmRegimeDetector`, wenn `hmm.enabled=true` UND eine trainierte Parameter-Datei auf Disk existiert. Der Detector wird ueber `RulesEngine(properties, diagnosticSink, hmmRegimeDetector)` an den `RegimeResolver` durchgereicht. Bei fehlender Parameter-Datei wird ein Warning geloggt und auf KPI-basiertes Regime zurueckgefallen.

## Offene Punkte
- Keine offenen Punkte. Alle DoD-Schritte abgeschlossen.

## Test-Ergebnisse

### Unit-Tests (80+ Tests)
- **RegimeProbabilitiesTest**: 16 Tests (Validierung, Randfaelle, NaN)
- **HmmParametersTest**: 18 Tests (Dimensionen, Row-Stochastik, Varianzen, Defensive Copies)
- **HmmRegimeDetectorTest**: 21 Tests (Forward-Algorithmus, Regime-Switch, Batch, Numerical Stability, Edge Cases)
- **HmmTrainerTest**: 11 Tests (Baum-Welch Konvergenz, Variance Floor, Validierung)
- **HmmParameterStoreTest**: 10 Tests (Round-Trip, Fehlerbehandlung, Overwrite)
- **HmmRegimeDetectionIntegrationTest**: 4 Tests (E2E Pipeline, RegimeResolver+HMM, Regime-Transition)

### Build-Ergebnisse
- odin-brain: 1170 Tests, 0 Failures
- odin-core: 296 Tests, 0 Failures
- odin-app: 305 Tests, 0 Failures
- odin-backtest: 349 Tests, 0 Failures

## ChatGPT-Sparring

### Edge-Cases identifiziert und implementiert:
1. **NaN Observation**: NaN propagiert durch Gaussian PDF und Log-Space zu NaN-Probabilities. Caller (RegimeResolver) muss NaN-Inputs abfangen. Test: `detect_nanObservation_doesNotThrow()`
2. **Absorbing State**: Transitions-Matrix mit Identity-Matrix (1.0 Diagonale). Wenn pi=[1,0,0], bleibt Detector in Bull trotz Bear-Observations. Test: `detect_absorbingState_locksIntoBull()`
3. **Fast Whipsaw**: Alternierende Bull/Bear-Observations bei engen Varianzen. Numerische Stabilitaet verifiziert (sum=1.0, alle finite). Test: `detect_fastWhipsaw_probabilitiesRemainValid()`
4. **Outlier Spike Recovery**: 10 Bull-Observations, 1 extremer Bear-Outlier (-0.1), 6 Bull-Recovery. Detector erholt sich dank sticky Transitions. Test: `detect_singleOutlierSpike_recoverAfterNormalObservations()`
5. **Reset Equivalence**: Reset + Detect muss identisch zu Fresh Instance sein. Test: `detect_resetThenDetect_matchesFreshInstance()`

## Gemini-Review

### Dimension 1: Code Quality / Numerical Stability -- BESTANDEN
- Forward-Algorithmus korrekt im Log-Space implementiert
- Log-Sum-Exp Trick korrekt angewendet
- Baum-Welch E-Step und M-Step mathematisch korrekt
- Variance Floor (1e-6) und Transition Floor (1e-6) ausreichend
- safeLog() und NEGATIVE_INFINITY Guards sauber implementiert

### Dimension 2: Concept Fidelity -- BESTANDEN
- 3-State-Mapping (Bull=TREND_UP, Neutral=RANGE_BOUND, Bear=TREND_DOWN) korrekt
- EMA9-EMA21-Spread als Return-Proxy valide und sinnvoll (modelliert Momentum statt roher Volatilitaet)
- Feature Flag korrekt implementiert (`hmmRegimeDetector != null && config.hmm().enabled()`)
- HMM ersetzt KPI-Regime komplett wenn aktiviert, Subregimes bleiben deterministisch
- LLM Secondary Filter funktioniert unabhaengig von KPI/HMM als Primary

### Dimension 3: Practical Regime-Switching Latency -- HINWEIS
- Forward-Algorithmus hat inhaerent Lag (nur Past-Data, kein Smoothing)
- Sticky Transition Matrix (0.8 Diagonale) impliziert erwartete 5 Perioden Verweildauer pro State
- Confirmation Lag in RegimeResolver addiert sich: HMM-Flip (3-4 Bars) + Confirmation (N Bars)
- Bei 5-Min-Chart: 30+ Minuten Gesamt-Latenz bei Regime-Wechsel moeglich
- HMM hat keinen dedizierten HIGH_VOLATILITY-State (nur 3 States)
- **Empfehlung**: Confirmation Lag reduzieren/eliminieren wenn HMM aktiv (HMM hat eigene Glaettung)
