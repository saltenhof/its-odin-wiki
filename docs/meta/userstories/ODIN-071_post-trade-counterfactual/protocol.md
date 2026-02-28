# Protokoll: ODIN-071 -- Post-Trade Counterfactual Analysis

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (8 in odin-brain, 7 in odin-backtest)
- [x] Integrationstests geschrieben (2 in odin-brain, 4 in odin-backtest)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Gemini-Review Dimension 1 (Code Quality)
- [x] Gemini-Review Dimension 2 (Architektur)
- [x] Gemini-Review Dimension 3 (Trading Domain)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Build & Test Status
- **305 Tests, 0 Failures, 0 Errors** (full reactor build)
- Alle 10 Maven-Module kompilieren sauber

## Design-Entscheidungen

### CounterfactualAction als eigenes Enum in odin-api
Die Story spezifiziert `CounterfactualDecision` als Record in `de.its.odin.api.dto`. Da odin-api nicht von odin-brain abhaengen darf, wurde ein dediziertes `CounterfactualAction` Enum in odin-api definiert (ENTRY, HOLD, EXIT), das die vereinfachte Semantik der Story abbildet.

### Counterfactual Logic: Quant-Only Determination
Fuer `quantOnlyAction`: Wenn QuantVote ALLOW_ENTRY/STRONG_ENTRY und alle Gates bestanden und RegimeConfidence >= Floor, dann ENTRY. Bei Rules-Engine EXIT-Signal, dann EXIT. Sonst HOLD.

### Counterfactual Logic: LLM-Only Determination
Fuer `llmOnlyAction`: Hard-Risk-Exits (Stop-Loss) aus der Rules-Engine ueberschreiben LLM-Entscheidungen (nach ChatGPT-Review-Finding). Wenn LlmVote EXIT_NOW mit offener Position, dann EXIT. Wenn LlmVote Entry-permitting mit Confidence >= Threshold, dann ENTRY. Sonst HOLD.

### FusionResult Backwards-Compatible Extension
FusionResult erhielt ein 12. Feld `@Nullable CounterfactualDecision counterfactual` mit backwards-kompatiblem 11-Parameter-Konstruktor und `withCounterfactual()` Copy-Methode.

### Configuration via BrainProperties
`CounterfactualProperties` nested Record mit `enabled` und `logInEvent` Booleans. Namespace: `odin.brain.counterfactual.*`. Defaults: beide true.

### CounterfactualAnalyzer in odin-backtest
Stateless Analyzer in `de.its.odin.backtest.counterfactual`. Gruppiert DecisionEventSnapshots nach Regime, berechnet per-Variant Returns/Win-Rates. Return-Attribution: ENTRY-Decisions erhalten den Return, HOLD/EXIT erhalten 0.

## ChatGPT-Sparring (Test-Review)

### Findings
1. **Placeholder LLM Votes (provider=NONE) nicht geprueft** -- evaluateLlmOnly prueft nicht `llmVote.placeholder()`. Risiko gering da Placeholder-Votes NO_TRADE Action tragen. Akzeptiert als bekannte Limitation.
2. **Hard-Risk-Exits fuer LLM-Only nicht erzwungen** -- evaluateLlmOnly ignorierte IntentType.EXIT aus Rules-Engine. **FIX EINGEARBEITET:** LLM-Only respektiert jetzt Stop-Loss/Forced-Close aus Rules-Engine.
3. **tradeCount Naming** -- VariantComparison.tradeCount ist eigentlich "decision count", nicht "trade count". Akzeptiert als Doku-Verbesserung fuer spaetere Story.
4. **EXIT-Timing nicht gemessen** -- Analyzer kann nicht quantifizieren ob LLM bessere Exits macht. Anerkannte Limitation, out of scope fuer Phase 1.
5. **Test-Luecken identifiziert** -- Gate-Cascade-Failure, Regime-Confidence-Boundary, Calibrated-Confidence-Path. Teilweise durch Integration-Tests abgedeckt, Rest als Future-Work notiert.

### Eingearbeitete Aenderungen
- evaluateLlmOnly: Hard-Risk-Exit (IntentType.EXIT) vor LLM-spezifischen Checks eingefuegt

## Gemini-Review (3 Dimensionen)

### Dimension 1: Code Quality
- **Positiv:** Strikte Einhaltung der Coding-Standards (keine `var`, explizite Typen, Records fuer DTOs, vollstaendige JavaDoc, Enums, defensive Programmierung mit Objects.requireNonNull)
- **Hinweis:** Manuelle JSON-Konstruktion in logFusion fragil -- pre-existing Issue, nicht durch ODIN-071 eingefuehrt

### Dimension 2: Architektur
- **Positiv:** Saubere Modul-Grenzen (odin-api fuer Contracts, odin-brain fuer Logic, odin-backtest fuer Analysis). CounterfactualAnalyzer korrekt stateless. FusionResult sauber erweitert.
- **Hinweis:** DecisionArbiter God-Class-Tendenz -- pre-existing, Counterfactual-Extraction als spaetere Verbesserung notiert

### Dimension 3: Trading Domain
- **Positiv:** Pragmatische Vereinfachungen fuer Intraday-Trading akzeptabel (gleiche Entry/Exit-Preise). Regime-Konservatismus korrekt.
- **Hinweis:** Return-Attribution nur fuer ENTRY (EXIT/HOLD = 0) kann LLM-Exit-Value nicht messen -- anerkannte Limitation, dokumentiert im Story-Scope

## Geaenderte Dateien

### Neue Dateien
- `odin-api/src/main/java/de/its/odin/api/model/CounterfactualAction.java`
- `odin-api/src/main/java/de/its/odin/api/dto/CounterfactualDecision.java`
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/VariantComparison.java`
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/CounterfactualReport.java`
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/CounterfactualAnalyzer.java`
- `odin-brain/src/test/java/de/its/odin/brain/arbiter/CounterfactualDecisionTest.java`
- `odin-brain/src/test/java/de/its/odin/brain/arbiter/CounterfactualDecisionIntegrationTest.java`
- `odin-backtest/src/test/java/de/its/odin/backtest/counterfactual/CounterfactualAnalyzerTest.java`
- `odin-backtest/src/test/java/de/its/odin/backtest/counterfactual/CounterfactualAnalyzerIntegrationTest.java`

### Modifizierte Dateien
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/FusionResult.java` -- +counterfactual Feld
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/DecisionArbiter.java` -- +counterfactual Computation
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` -- +CounterfactualProperties
- `odin-brain/src/main/resources/odin-brain.properties` -- +counterfactual Defaults
- `odin-brain/src/test/java/de/its/odin/brain/TestBrainProperties.java` -- +counterfactual Parameter
- `odin-backtest/src/main/java/de/its/odin/backtest/BacktestReport.java` -- +counterfactualReport Feld
- `odin-app/src/main/java/de/its/odin/app/service/ParameterOverrideApplier.java` -- +counterfactual Param
- Diverse Test-Dateien: BrainProperties-Konstruktor-Updates (KpiEngineTest, GateCascadeEvaluatorIntegrationTest, PipelineFactoryTest, BacktestRunnerTest, BacktestPipelineIntegrationTest, BacktestRunnerGovernanceIntegrationTest, WalkForwardRunnerIntegrationTest, ParameterOverrideApplierTest)
