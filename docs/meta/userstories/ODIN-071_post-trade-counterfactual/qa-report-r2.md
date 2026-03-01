# QA Report: ODIN-071 — Post-Trade Counterfactual Analysis
**Runde:** 2
**Datum:** 2026-02-28
**Ergebnis:** PASS

---

## M1-Remediation: BacktestRunner-Integration — BEHOBEN

### Befund Runde 1
`BacktestRunner` integrierte `CounterfactualAnalyzer` nicht. `BacktestReport.counterfactualReport` war immer `null`.

### Nachweis der Behebung

**Commit:** `19dd07e` — `fix(counterfactual): ODIN-071 R2 — wire CounterfactualAnalyzer into BacktestRunner`

**Geänderte Dateien:**
- `odin-backtest/src/main/java/de/its/odin/backtest/BacktestRunner.java` (+114 Zeilen)
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/CounterfactualCapturingEventLog.java` (neu, 176 Zeilen)
- `odin-backtest/src/test/.../CounterfactualBacktestIntegrationTest.java` (neu, 268 Zeilen)
- `odin-backtest/src/test/.../CounterfactualCapturingEventLogTest.java` (neu, 270 Zeilen)

**Architekturlösung:**
Analoges Decorator-Pattern wie `TcaCapturingEventLog` (ODIN-064):

1. `CounterfactualCapturingEventLog` — EventLog-Decorator, der FUSION-Events mit counterfactual-Feldern abfängt und als `DecisionEventSnapshot` parsed
2. `BacktestRunner.runSingleDay` — wraps EventLog-Kette mit `CounterfactualCapturingEventLog`, sammelt Snapshots nach jedem Tag
3. `BacktestRunner.runWithOverrides` — ruft `runCounterfactualAnalysis()` nach Robustness-Analyse auf, reichert Snapshots mit Durchschnittsreturn an, übergibt `CounterfactualReport` an `BacktestReport` (10-Parameter-Konstruktor)

**Kritische Code-Stellen:**

```java
// BacktestRunner.java (Zeile 340) — Snapshot-Akkumulator
List<CounterfactualAnalyzer.DecisionEventSnapshot> counterfactualSnapshots = new ArrayList<>();

// Zeile 388-398 — Analyse und Report-Konstruktion
CounterfactualReport counterfactualReport = null;
if (overrideBrainProps.counterfactual().enabled() && !counterfactualSnapshots.isEmpty()
        && !cancelled.get()) {
    counterfactualReport = runCounterfactualAnalysis(batchId, counterfactualSnapshots, dailyResults);
}
BacktestReport report = new BacktestReport(batchId, startDate, endDate, barInterval,
        instruments, governanceVariant, dailyResults, summary, robustnessReport,
        counterfactualReport);  // 10-Parameter-Konstruktor mit counterfactualReport

// Zeile 501-503 — EventLog-Dekorator-Kette
TcaCapturingEventLog tcaCapturingEventLog = new TcaCapturingEventLog(baseEventLog);
CounterfactualCapturingEventLog cfCapturingEventLog =
        new CounterfactualCapturingEventLog(tcaCapturingEventLog);
```

**Kritischer Integrationstest:**
`CounterfactualBacktestIntegrationTest.backtestWithFusionEvents_producesNonNullCounterfactualReport()` verifiziert explizit, dass `BacktestReport.counterfactualReport != null` nach einem Backtest mit FUSION-Events.

---

## Build-Verifikation

```
odin-backtest (Surefire): 369 Tests, 0 Failures, 0 Errors
odin-backtest (Failsafe):  50 Tests, 0 Failures, 0 Errors
odin-brain (Surefire):   1289 Tests, 0 Failures, 0 Errors
BUILD SUCCESS für odin-api, odin-brain, odin-backtest
```

**Pre-existing Failures (nicht durch ODIN-071 verursacht):**
- `odin-execution` — `TrailingStopManagerIntegrationTest.trailingStopEventIsLoggedOnUpdate` (pre-existing payload-Format-Mismatch)
- `odin-execution` — Kompilierungsfehler `PreTradeProperties` in `PreTradeControlGate` (pre-existing, durch andere Story eingeführt)

Diese Failures existierten bereits in Runde 1 und wurden im protocol.md explizit dokumentiert.

---

## 2.1 Code-Qualität — PASS

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Kein `var` | PASS | Keine `var`-Verwendung in neuen Klassen |
| Keine Magic Numbers | PASS | String-Konstanten als `private static final` (`FUSION_EVENT_TYPE`, `FUSED_REGIME_KEY` etc.) |
| Records für DTOs | PASS | `CounterfactualDecision`, `CounterfactualReport`, `VariantComparison` als Records |
| ENUM statt String | PASS | `CounterfactualAction` als Enum in odin-api |
| JavaDoc | PASS | Alle neuen public Klassen, Methoden und Felder mit JavaDoc |
| Keine TODO/FIXME | PASS | Keine verbleibenden TODOs |
| Code-Sprache: Englisch | PASS | Code und Kommentare auf Englisch |
| Namespace-Konvention | PASS | `de.its.odin.backtest.counterfactual`, `odin.brain.counterfactual.*` |
| Port-Abstraktion | PASS | `EventLog`-Interface korrekt implementiert als Decorator |

Neue Dateien Runde 2:
- `odin-backtest/src/main/java/de/its/odin/backtest/counterfactual/CounterfactualCapturingEventLog.java` — korrekt
- `odin-backtest/src/test/.../CounterfactualCapturingEventLogTest.java` — korrekt
- `odin-backtest/src/test/.../CounterfactualBacktestIntegrationTest.java` — korrekt

---

## 2.2 Unit-Tests — PASS

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Unit-Tests für neue Geschäftslogik | PASS | 13 Tests in `CounterfactualCapturingEventLogTest` (Payload-Parsing, FusionAction-Mapping, Edge Cases) |
| Unit-Tests für CounterfactualAnalyzer | PASS | 7 Tests in `CounterfactualAnalyzerTest` (Runde 1, unverändert) |
| Unit-Tests für DecisionArbiter Counterfactual | PASS | 8 Tests in `CounterfactualDecisionTest` (Runde 1, unverändert) |
| Namenskonvention `*Test` | PASS | Korrekt |
| Mocks/Stubs für Port-Interfaces | PASS | `NO_OP_DELEGATE` Lambda-Implementierung des `EventLog`-Interface |
| AC: Quant ENTRY + LLM HOLD → Hybrid HOLD | PASS | `fusionEvent_withCounterfactual_capturedAsSnapshot()` verifiziert Mapping |

Gesamtzahl Surefire-Tests odin-backtest: **369, 0 Failures**

---

## 2.3 Integrationstests — PASS

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Integrationstest BacktestRunner E2E | PASS | `CounterfactualBacktestIntegrationTest` — 4 Tests, 0 Failures |
| Kritische Assertion: `counterfactualReport != null` | PASS | `backtestWithFusionEvents_producesNonNullCounterfactualReport()` explizit |
| LLM Value-Add End-to-End | PASS | `llmVetoesLosingTrade_positiveValueAddEndToEnd()` |
| Multi-Day Accumulation | PASS | `multiDayAccumulation_snapshotsFromAllDaysCombined()` |
| Integrationstest CounterfactualAnalyzer | PASS | 4 Tests in `CounterfactualAnalyzerIntegrationTest` (Runde 1) |
| Integrationstest DecisionArbiter | PASS | 2 Tests in `CounterfactualDecisionIntegrationTest` (Runde 1) |
| Namenskonvention `*IntegrationTest` | PASS | Korrekt (Failsafe) |

Gesamtzahl Failsafe-Tests odin-backtest: **50, 0 Failures**

---

## 2.4 DB-Tests — N/A

Keine Datenbankzugriffe in ODIN-071 — kein DB-Schema, keine Repositories. Kriterium nicht anwendbar.

---

## 2.5 ChatGPT-Sparring — PASS

Dokumentiert in `protocol.md`, Abschnitt "ChatGPT-Sparring":
- 5 Findings dokumentiert
- **Finding 2 ("Hard-Risk-Exits für LLM-Only nicht erzwungen") eingearbeitet** — `evaluateLlmOnly()` respektiert jetzt Stop-Loss/Forced-Close
- Findings 1, 3, 4, 5 bewertet und mit Begründung akzeptiert/als Future-Work notiert

---

## 2.6 Gemini-Review (3 Dimensionen) — PASS

Dokumentiert in `protocol.md`, Abschnitt "Gemini-Review":
- **Dimension 1 (Code):** Positiv + Hinweis auf pre-existing manuelle JSON-Konstruktion
- **Dimension 2 (Architektur):** Positiv + Hinweis auf pre-existing God-Class-Tendenz in DecisionArbiter
- **Dimension 3 (Trading Domain):** Positiv + Return-Attribution-Limitation dokumentiert

Alle Findings bewertet, keine kritischen unbehobenen Issues.

---

## 2.7 Protokolldatei — PASS

`protocol.md` enthält alle Pflichtabschnitte:
- [x] Working State (alle Schritte abgehakt, inkl. "Runde 2 Remediation (M1)")
- [x] Build & Test Status (369+50 Tests, 0 Failures; pre-existing Failures dokumentiert)
- [x] Design-Entscheidungen (5 Entscheidungen mit Begründung, +R2 Return-Enrichment-Design)
- [x] ChatGPT-Sparring (5 Findings, 1 eingearbeitet)
- [x] Gemini-Review (3 Dimensionen dokumentiert)
- [x] **Runde 2 Remediation-Abschnitt** (Problem, Lösung, Design-Entscheidung, neue Tests)
- [x] Geänderte Dateien (vollständige Liste für Runde 1 und Runde 2)

---

## 2.8 Abschluss — PASS

- Backend: Commit `19dd07e` — `fix(counterfactual): ODIN-071 R2 — wire CounterfactualAnalyzer into BacktestRunner`
- Wiki: Commit `c39574e` — `docs(ODIN-071): update protocol with R2 remediation details`
- Beide Repos auf Remote gepusht (Branch up to date)
- Story-Verzeichnis enthält `story.md` + `protocol.md`

---

## Zusammenfassung

| DoD-Punkt | Status |
|-----------|--------|
| 2.1 Code-Qualität | PASS |
| 2.2 Unit-Tests | PASS |
| 2.3 Integrationstests | PASS |
| 2.4 DB-Tests | N/A |
| 2.5 ChatGPT-Sparring | PASS |
| 2.6 Gemini-Review | PASS |
| 2.7 Protokolldatei | PASS |
| 2.8 Commit + Push | PASS |

**M1-Remediation: BEHOBEN**

**Gesamtergebnis: PASS**

`BacktestRunner` integriert `CounterfactualAnalyzer` vollständig in die Backtest-Pipeline. Die `CounterfactualCapturingEventLog`-Klasse fängt FUSION-Events mit counterfactual-Feldern während der Simulation ab. Nach dem Backtest-Lauf werden die Snapshots mit Durchschnittsreturns angereichert und durch `CounterfactualAnalyzer.analyze()` ausgewertet. Der resultierende `CounterfactualReport` ist Teil des `BacktestReport` (10-Parameter-Konstruktor). Der kritische Integrationstest `CounterfactualBacktestIntegrationTest.backtestWithFusionEvents_producesNonNullCounterfactualReport()` verifiziert explizit, dass `BacktestReport.counterfactualReport != null` wenn FUSION-Events mit counterfactual-Daten während des Backtests emittiert wurden. Alle 419 Tests in odin-backtest bestehen.
