# QA Report: ODIN-071 — Post-Trade Counterfactual Analysis
**Runde:** 1
**Datum:** 2026-02-28
**Ergebnis:** FAIL

---

## Build-Verifikation

```
BUILD SUCCESS — alle 10 Maven-Module kompilieren sauber
Laufzeit: 31s
```

Tests laufen:
- `CounterfactualDecisionTest`: **8 Tests, 0 Failures** (odin-brain, Surefire)
- `CounterfactualAnalyzerTest`: **7 Tests, 0 Failures** (odin-backtest, Surefire)
- `CounterfactualDecisionIntegrationTest`: **2 Tests, 0 Failures** (odin-brain, Failsafe)
- `CounterfactualAnalyzerIntegrationTest`: **4 Tests, 0 Failures** (odin-backtest, Failsafe)

---

## 2.1 Code-Qualität — PASS

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Kein `var` | PASS | Keine `var`-Verwendung in neuen Klassen gefunden |
| Keine Magic Numbers | PASS | `REGIME_CONFIDENCE_FLOOR` und andere Konstanten als `private static final` |
| Records für DTOs | PASS | `CounterfactualDecision`, `CounterfactualReport`, `VariantComparison` sind Records |
| ENUM statt String | PASS | `CounterfactualAction` als Enum in odin-api |
| JavaDoc | PASS | Alle neuen public Klassen, Methoden und Felder haben JavaDoc |
| Keine TODO/FIXME | PASS | Keine verbleibenden TODOs gefunden |
| Code-Sprache: Englisch | PASS | Code und Kommentare durchgängig auf Englisch |
| Namespace-Konvention | PASS | `odin.brain.counterfactual.*` in properties, Package `de.its.odin.backtest.counterfactual` |
| Port-Abstraktion | PASS | `EventLog`-Interface wird verwendet, nicht konkrete Impl |

Neue Dateien:
- `odin-api/src/main/java/de/its/odin/api/model/CounterfactualAction.java` — korrekt
- `odin-api/src/main/java/de/its/odin/api/dto/CounterfactualDecision.java` — korrekt
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/VariantComparison.java` — korrekt
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/CounterfactualReport.java` — korrekt
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/CounterfactualAnalyzer.java` — korrekt

---

## 2.2 Unit-Tests — PASS

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Unit-Tests für neue Geschäftslogik | PASS | 8 Tests in `CounterfactualDecisionTest`, 7 Tests in `CounterfactualAnalyzerTest` |
| Namenskonvention `*Test` | PASS | Korrekt |
| Mocks/Stubs für Port-Interfaces | PASS | `TestEventLog` (innere Klasse) implementiert `EventLog`-Interface ohne Spring-Kontext |
| Spezifisches AC abgedeckt: Quant ENTRY + LLM HOLD → Hybrid HOLD | PASS | `counterfactual_quantEntry_llmHold_hybridHold()` abgedeckt |
| `CounterfactualAnalyzer` mit synthetischen Daten | PASS | Mehrere Szenarien: Multi-Regime, Win-Rate, positive/negative LLM ValueAdd |

---

## 2.3 Integrationstests — PASS

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Integrationstest für DecisionArbiter E2E | PASS | `CounterfactualDecisionIntegrationTest`: 5-Decision-Sequenz durch echten GateCascadeEvaluator |
| Integrationstest für CounterfactualAnalyzer | PASS | `CounterfactualAnalyzerIntegrationTest`: 20-Decision-Szenario + 100-Event-Skalierungstest |
| Namenskonvention `*IntegrationTest` | PASS | Korrekt (Failsafe) |
| Min. 1 Integrationstest pro Story | PASS | 2 in odin-brain + 4 in odin-backtest = 6 gesamt |

---

## 2.4 DB-Tests — N/A

Keine Datenbankzugriffe in ODIN-071 — kein DB-Schema, keine Repositories. AC nicht anwendbar.

---

## 2.5 ChatGPT-Sparring — PASS

Dokumentiert in `protocol.md`, Abschnitt "ChatGPT-Sparring":
- 5 Findings dokumentiert
- **Finding 2 ("Hard-Risk-Exits für LLM-Only nicht erzwungen") eingearbeitet** — `evaluateLlmOnly()` respektiert jetzt Stop-Loss/Forced-Close
- Findings 1, 3, 4, 5 bewertet und mit Begründung akzeptiert/als Future-Work notiert

---

## 2.6 Gemini-Review (3 Dimensionen) — PASS

Dokumentiert in `protocol.md`, Abschnitt "Gemini-Review":

- **Dimension 1 (Code):** Positiv + Hinweis auf pre-existing manuelle JSON-Konstruktion (nicht durch ODIN-071 eingeführt)
- **Dimension 2 (Architektur):** Positiv + Hinweis auf pre-existing God-Class-Tendenz in DecisionArbiter
- **Dimension 3 (Trading Domain):** Positiv + Return-Attribution-Limitation dokumentiert

Alle Findings bewertet, keine kritischen unbehobenen Issues.

---

## 2.7 Protokolldatei — PASS

`protocol.md` enthält alle Pflichtabschnitte:
- [x] Working State (alle Schritte abgehakt)
- [x] Build & Test Status (305 Tests, 0 Failures)
- [x] Design-Entscheidungen (5 Entscheidungen mit Begründung)
- [x] ChatGPT-Sparring (5 Findings, 1 eingearbeitet)
- [x] Gemini-Review (3 Dimensionen dokumentiert)
- [x] Geänderte Dateien (vollständige Liste)

---

## 2.8 Abschluss — PASS

- Backend: Commit `344efac` — `feat(counterfactual): ODIN-071 — Post-Trade Counterfactual Analysis`
- Wiki: Commit `8f8ae5a` — `docs(ODIN-071): add implementation protocol`
- Beide Repos auf Remote gepusht (`branch is up to date with 'origin/main'`)
- Story-Verzeichnis enthält `story.md` + `protocol.md`

---

## KRITISCHER MANGEL — M1

### M1: BacktestRunner integriert CounterfactualAnalyzer NICHT

**Schweregrad:** KRITISCH — Akzeptanzkriterium nicht erfüllt

**Story AC:** `BacktestRunner` soll nach dem Backtest-Lauf `CounterfactualAnalyzer` ausführen, wenn EventLog-Daten vorhanden sind.

**Technische Details (story.md):**
> `BacktestRunner` (Änderung): Nach Backtest-Lauf: CounterfactualAnalyzer ausführen wenn EventLog-Daten vorhanden

**Befund im Code:**

`BacktestRunner.java` (Zeilen 377-388):
```java
// Run robustness analysis as post-processing step when enabled
RobustnessReport robustnessReport = null;
if (robustnessProperties.enabled() && !cancelled.get()) {
    robustnessReport = runRobustnessAnalysis(batchId, dailyResults, summary);
}

BacktestReport report = new BacktestReport(batchId, startDate, endDate, barInterval,
        instruments, governanceVariant, dailyResults, summary, robustnessReport);
```

Der `BacktestRunner` verwendet den 9-Parameter-Konstruktor ohne `counterfactualReport`. `CounterfactualAnalyzer` wird nirgendwo in `BacktestRunner` importiert oder aufgerufen. `BacktestReport.counterfactualReport` ist immer `null`.

**Protokoll-Aussage in `protocol.md`:**
> `BacktestRunner` Änderung ist als geänderte Datei gelistet

Aber `git show HEAD -- odin-backtest/src/main/java/de/its/odin/backtest/BacktestRunner.java` zeigt, dass `BacktestRunner.java` NICHT in den geänderten Dateien des Commits enthalten ist. Die Datei wurde nicht geändert.

**Auswirkung:**
- `CounterfactualReport` wird nach keinem Backtest produziert
- `BacktestReport.counterfactualReport` ist immer null
- Die Kernfunktionalität der Story (Counterfactual-Analyse nach Backtest) ist nicht nutzbar
- Die zugehörigen Integrationstest (`CounterfactualAnalyzerIntegrationTest`) testen nur den `CounterfactualAnalyzer` isoliert, nicht die BacktestRunner-Integration

**Erforderliche Nacharbeit:**
1. `BacktestRunner` muss `CounterfactualAnalyzer` nach dem Backtest-Lauf aufrufen (analog zu `runRobustnessAnalysis`)
2. Decision-Events müssen aus dem EventLog extrahiert werden (bzw. ein dedizierter Collector für `DecisionEventSnapshot`-Objekte muss während des Laufs gefüllt werden)
3. Ein Integrationstest für `BacktestRunner` muss verifizieren, dass `BacktestReport.counterfactualReport != null` nach einem Backtest mit Decision-Events

---

## Zusammenfassung

| DoD-Punkt | Status |
|-----------|--------|
| 2.1 Code-Qualität | PASS |
| 2.2 Unit-Tests | PASS |
| 2.3 Integrationstests | PASS (aber BacktestRunner-Integration fehlt) |
| 2.4 DB-Tests | N/A |
| 2.5 ChatGPT-Sparring | PASS |
| 2.6 Gemini-Review | PASS |
| 2.7 Protokolldatei | PASS |
| 2.8 Commit + Push | PASS |

**Gesamtergebnis: FAIL**

Ein kritisches Akzeptanzkriterium ist nicht erfüllt: `BacktestRunner` integriert `CounterfactualAnalyzer` nicht. Die Klasse existiert, ist korrekt implementiert und gut getestet — aber die Verdrahtung in die BacktestPipeline fehlt. Das entspricht dem ZERO DEBT RULE-Verstoß: die wesentliche End-to-End-Funktionalität ist nicht vollständig.
