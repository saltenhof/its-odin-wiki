# ODIN-071: Post-Trade Counterfactual Analysis

## Modul

odin-brain, odin-api, odin-backtest

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

M

## Kontext

ODIN fusioniert Quant- und LLM-Signale im `DecisionArbiter`. Ohne systematisches Tracking ist unbekannt, ob der LLM echten Mehrwert liefert oder nur Komplexitaet hinzufuegt. Diese Story fuehrt Counterfactual-Logging ein: Pro Entscheidung wird zusaetzlich persistiert, was Quant-Only und LLM-Only entschieden haetten. Ein Analyse-Tool vergleicht die Ergebnisse pro Regime und berechnet den LLM-Mehrwert. Das ist die essentielle Grundlage fuer die spaetere regime-abhaengige Arbiter-Gewichtung (Thema 12) und adaptive Algorithmen (Thema 22).

## Scope

### In Scope

- **Counterfactual-Logging im DecisionArbiter:** Pro Entscheidung zusaetzlich `quantOnlyDecision` und `llmOnlyDecision` loggen
- **Counterfactual-Felder im EventLog:** Was haette Quant allein entschieden? Was haette LLM allein entschieden?
- **`CounterfactualAnalyzer`:** Analyse-Tool fuer Backtest-Daten, vergleicht Quant-Only vs. Hybrid vs. LLM-Only Return pro Regime
- **Aggregierte Metriken:** Win-Rate, Avg Return, Sharpe pro Strategie-Variante (Quant-Only, LLM-Only, Hybrid), aufgeschluesselt nach Regime
- **Integration in BacktestReport:** Counterfactual-Vergleichsmetriken als Teil des Reports

### Out of Scope

- Shapley-Value-Attribution (mathematisch korrekte Wertbeitrags-Zuordnung) — spaetere Erweiterung
- Live-Counterfactual-Dashboard (Frontend-Story)
- Automatische Gewichtsanpassung basierend auf Counterfactual-Ergebnissen (Thema 12)
- Paper-Trading-Modus fuer Quant-Only-Vergleich (zu komplex)

## Akzeptanzkriterien

- [ ] `FusionResult` (`de.its.odin.brain.arbiter.FusionResult`) enthaelt neue Felder: `quantOnlyAction` (Action), `llmOnlyAction` (Action) — was haette jede Seite allein entschieden?
- [ ] `DecisionArbiter` bestimmt `quantOnlyAction`: Wuerde Quant allein (ohne LLM-Input) Entry/Hold/Exit empfehlen? Basierend auf Gate-Kaskade + Quant-Regeln
- [ ] `DecisionArbiter` bestimmt `llmOnlyAction`: Wuerde LLM allein (ohne Quant-Veto) Entry/Hold/Exit empfehlen? Basierend auf LlmTacticalOutput
- [ ] Counterfactual-Felder werden im EventLog persistiert (als Teil des Decision-Event-Payloads)
- [ ] `CounterfactualAnalyzer` aggregiert pro Regime: Hybrid-Return vs. Quant-Only-Return vs. LLM-Only-Return
- [ ] `CounterfactualReport` Record mit: `regimeBreakdown` (Map<Regime, VariantComparison>), `overallLlmValueAdd` (double)
- [ ] Unit-Test verifiziert: Wenn Quant ENTRY empfiehlt, LLM HOLD empfiehlt und Hybrid HOLD entscheidet (LLM-Veto), dann `quantOnlyAction=ENTRY`, `llmOnlyAction=HOLD`
- [ ] Unit-Test verifiziert: `CounterfactualAnalyzer` berechnet korrekte aggregierte Returns fuer synthetische Daten
- [ ] Counterfactual-Logging verursacht keine messbaren Performance-Kosten (< 1ms pro Entscheidung)

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `CounterfactualAnalyzer` | odin-backtest | `de.its.odin.backtest.counterfactual` | Analysiert Decision-Events aus EventLog. Input: List<DecisionEventPayload>. Output: CounterfactualReport |
| `CounterfactualReport` | odin-backtest | `de.its.odin.backtest.counterfactual` | Record: overallLlmValueAdd, regimeBreakdown (Map<Regime, VariantComparison>) |
| `VariantComparison` | odin-backtest | `de.its.odin.backtest.counterfactual` | Record: hybridReturn, quantOnlyReturn, llmOnlyReturn, hybridWinRate, quantOnlyWinRate, llmOnlyWinRate, tradeCount |
| `CounterfactualDecision` | odin-api | `de.its.odin.api.dto` | Record: quantOnlyAction (Action enum), llmOnlyAction (Action enum) |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `FusionResult` (`de.its.odin.brain.arbiter.FusionResult`) | Neue Felder: `CounterfactualDecision counterfactual` (optional, nullable) |
| `DecisionArbiter` (`de.its.odin.brain.arbiter.DecisionArbiter`) | Vor der eigentlichen Fusion: `quantOnlyAction` und `llmOnlyAction` separat berechnen. In FusionResult einsetzen |
| `DecisionArbiter` (Event-Logging-Abschnitt) | Counterfactual-Felder in JSON-Payload des Decision-Events aufnehmen |
| `BacktestReport` (`de.its.odin.backtest.BacktestReport`) | Optionales Feld `CounterfactualReport counterfactual` |
| `BacktestRunner` (`de.its.odin.backtest.BacktestRunner`) | Nach Backtest-Lauf: CounterfactualAnalyzer ausfuehren wenn EventLog-Daten vorhanden |

### Counterfactual-Logik im Arbiter

```
// Quant-Only: Was haette Quant ohne LLM entschieden?
quantOnlyAction = evaluateQuantOnly(gateCascadeResult, quantVote)
  - Wenn alle Gates PASS und QuantVote.regime == TREND_UP -> ENTRY
  - Wenn Position offen und QuantExitSignal -> EXIT
  - Sonst -> HOLD

// LLM-Only: Was haette LLM ohne Quant-Veto entschieden?
llmOnlyAction = evaluateLlmOnly(llmVote)
  - Wenn llmVote.action == ENTRY und llmVote.confidence > threshold -> ENTRY
  - Wenn llmVote.action == EXIT -> EXIT
  - Sonst -> HOLD
```

### Konfiguration

```properties
odin.brain.counterfactual.enabled=true
odin.brain.counterfactual.log-in-event=true
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 8: Post-Trade Counterfactual-Analyse, Abschnitte IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/06-rules-engine.md` — DecisionArbiter, Dual-Key-Logik, FusionResult
- `docs/backend/architecture/05-llm-integration.md` — LlmTacticalOutput, LlmVote

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, Events sind immutable, Records fuer DTOs

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
- [ ] Namespace-Konvention: `odin.brain.counterfactual.*`, `odin.backtest.counterfactual.*`

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `CounterfactualAnalyzer` (synthetische Decision-Events)
- [ ] Unit-Tests fuer Quant-Only und LLM-Only Logik im DecisionArbiter
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer EventLog

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: DecisionArbiter mit Counterfactual-Logging End-to-End
- [ ] Integrationstest: CounterfactualAnalyzer mit realistischen Event-Daten
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen und Akzeptanzkriterien
- [ ] Edge Cases diskutiert (keine LLM-Antwort, QUANT_ONLY-Modus, Regime-Wechsel mid-Trade)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Korrektheit der Counterfactual-Logik, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (Counterfactual-Definition, Regime-Breakdown)
- [ ] Dimension 3: Praxis-Review (Ist Quant-Only-Simulation realistisch ohne Paper-Trading?)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Counterfactual ist keine Simulation:** Wir loggen was die Einzelkomponenten entschieden HAETTEN, fuehren aber keine separate Simulation durch. Das ist eine Approximation — in der Realitaet wuerde Quant-Only andere Fills bekommen (andere Timing, andere Stops). Fuer eine erste Analyse ist diese Approximation ausreichend.
- **Performance-Impact:** Die zusaetzliche Berechnung von quantOnlyAction und llmOnlyAction ist vernachlaessigbar (Schwellenvergleiche, keine IO). Das Logging in den Event-Payload fuegt wenige Bytes hinzu.
- **Existing FusionResult.winnerSide:** Das bestehende `winnerSide`-Feld zeigt bereits, welche Seite "gewonnen" hat. Counterfactual ergaenzt das um die Frage "Was waere OHNE die andere Seite passiert?".
- **Return-Attribution:** `CounterfactualAnalyzer` kann den Return eines Trades nur der Hybrid-Entscheidung zuordnen. Fuer Quant-Only und LLM-Only ist der Return hypothetisch (Entry-Preis waere identisch, aber Exit-Timing koennte abweichen). Vereinfachung: gleicher Exit-Preis angenommen.
- **Regime-Breakdown ist der Kern:** Der Gesamt-LLM-ValueAdd kann positiv sein, waehrend der LLM in bestimmten Regimes schadet. Die Aufschluesselung nach Regime ist daher wichtiger als die Gesamtzahl.
