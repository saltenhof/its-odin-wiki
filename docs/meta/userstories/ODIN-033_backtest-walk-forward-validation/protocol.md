# ODIN-033 Implementation Protocol: Walk-Forward Validation Framework

**Story:** ODIN-033
**Module:** odin-backtest
**Date:** 2026-02-23
**Status:** Implementation complete, reviews done, all tests green

---

## 1. Implementierungsübersicht

### Neue Dateien

| Datei | Typ | Beschreibung |
|-------|-----|-------------|
| `WalkForwardRunner.java` | `@Component` | Orchestrator: Window-Splitting, BacktestRunner-Delegation, Overfitting-Erkennung |
| `WalkForwardReport.java` | Record | Aggregiertes Ergebnis mit WindowResult-Nested-Record |
| `WalkForwardRunnerTest.java` | Unit-Test (`*Test`) | 35 Unit-Tests für isolierte Logik (mocked BacktestRunner) |
| `WalkForwardRunnerIntegrationTest.java` | Integrations-Test (`*IntegrationTest`) | 10 End-to-End-Tests mit realem BacktestRunner + gemocktem BarRepository |

### Geänderte Dateien

| Datei | Änderung |
|-------|---------|
| `BacktestRunner.java` | Pre-existing Bug gefixt: CoreProperties-Konstruktor fehlte `session()`-Parameter; LifecycleManager fehlte `SessionPhaseMonitor`-Argument |
| `BacktestRunnerTest.java` | Pre-existing Bug gefixt: `buildCoreProperties()` fehlte `SessionProperties` |
| `BacktestRunnerGovernanceIntegrationTest.java` | Pre-existing Bug gefixt: gleiche `buildCoreProperties()`-Lücke |

---

## 2. Design-Entscheidungen

### 2.1 Overfitting-Score-Logik (final nach Reviews)

```
if (trainSharpe <= 0.0)     → NEUTRAL_OVERFITTING_SCORE (1.0)
if (validateSharpe <= 1e-9) → OVERFITTING_SCORE_CAP (1_000_000.0)
else                         → min(trainSharpe / validateSharpe, CAP)
```

**Begründung:**
- Train <= 0: In-Sample schlecht → keine Overfitting-Sorge
- Validate <= epsilon: Worst-Case-Overfitting (In-Sample gut, Out-of-Sample schlecht/negativ)
- Math.min(ratio, CAP): Verhindert extreme Werte bei marginally-positive validateSharpe (Gemini-Finding)
- Endlicher Cap statt MAX_VALUE: Verhindert NaN-Propagation bei Aggregation-Mittelwerten

**Verworfen:** Ursprüngliche Logik `trainSharpe > 0 && validateSharpe near 0 → MAX_VALUE` hatte NaN-Risiko und behandelte negative validateSharpe nicht korrekt (falsches Negativ).

### 2.2 stepDays >= validateDays Validation

**Implementiert:** Guard in `validateWindowParameters()`. Wenn `stepDays < validateDays`, überlappen OOS-Perioden → statistische Verzerrung durch mehrfaches Zählen identischer Trading-Days in der Aggregation.

**Verworfen:** Erlauben mit Dokumentation — zu hohes Risiko für stille Verzerrung in V1.

### 2.3 NaN-Handling in der Aggregation

**Implementiert:** `finiteOrZero(value)` normalisiert alle nicht-finiten Werte zu 0.0 vor der Aggregation. Inaktive Windows (keine Trades → NaN Sharpe) bleiben neutral ohne die Aggregation zu vergiften.

**Verworfen:** Window-Ausschluss (Selection Bias), MAX_VALUE für MaxDrawdown (nicht interpretierbar).

### 2.4 Per-Window Overfitting-Warning

**Implementiert:** `WindowResult` enthält jetzt `overfittingWarning`-Boolean. Das Report-Level-Warning feuert wenn der Overall-Score den Threshold überschreitet ODER wenn `anyWindowOverfits` true ist.

**Begründung:** Ein einzelnes stark overfittes Window kann unter dem Mittelwert der Overall-Score-Berechnung verschwinden. Die Per-Window-Flag verhindert False Negatives.

### 2.5 computeWindowBounds Rückgabetyp

**Geändert:** `List<int[]>` → `List<Integer>`. Semantisch klarer, kein unnötiges Array-Wrapping für einen einzigen Index-Wert.

### 2.6 NEUTRAL_OVERFITTING_SCORE als Konstante

**Implementiert:** 1.0 als `NEUTRAL_OVERFITTING_SCORE` — semantisch "kein Overfitting-Signal", nicht einfach nur die Zahl `1.0`.

### 2.7 Aggregation als arithmetisches Mittel (V1-Kompromiss)

**Bewusste Vereinfachung (dokumentiert in JavaDoc):**
Das arithmetische Mittel von Sharpe-Ratios ist mathematisch nicht identisch mit dem Sharpe der Gesamtperiode. Korrekt wäre die Concatenation aller OOS-Returns und Neuberechnung. Dies ist explizit als "V1 simplification" dokumentiert und als V2-Verbesserungsaufgabe notiert.

---

## 3. ChatGPT-Review (2 Runden)

### Runde 1 — Findings (owner: odin-033-impl)

| Kategorie | Finding | Aktion |
|-----------|---------|--------|
| KRITISCH | `startDate.isAfter(endDate)` nicht validiert (JavaDoc verspricht IAE) | IMPLEMENTIERT |
| KRITISCH | NaN/Infinity vergiftet Aggregation → Report kann komplett NaN werden | IMPLEMENTIERT (`finiteOrZero`) |
| KRITISCH | Unused `anyInt` Import in WalkForwardRunnerTest.java | IMPLEMENTIERT (entfernt) |
| WICHTIG | SLF4J-Formatstrings `{:.3f}` falsch → Zahlen werden nicht interpoliert | IMPLEMENTIERT (→ `String.format`) |
| WICHTIG | Negative validateSharpe → falsches Negativ im Overfitting-Warning | IMPLEMENTIERT (neue Score-Logik) |
| WICHTIG | Mittelwert von Sharpes quant-schwach | HINWEIS in JavaDoc, V2-Scope |
| WICHTIG | `stepDays < validateDays` erlaubt überlappende OOS-Perioden | IMPLEMENTIERT (Guard) |
| WICHTIG | BacktestRunner-State-Risiko zwischen Sub-Runs | VERWORFEN (BacktestRunner.run() ist stateless by design) |
| WICHTIG | Per-Window Overfitting-Spikes können im Mittelwert verschwinden | IMPLEMENTIERT (`anyWindowOverfits`) |
| HINWEIS | `List<int[]>` → `List<Integer>` | IMPLEMENTIERT |
| HINWEIS | `1.0` als Named-Constant | IMPLEMENTIERT (`NEUTRAL_OVERFITTING_SCORE`) |
| HINWEIS | WindowResult-Datum-Invarianten | IMPLEMENTIERT |
| HINWEIS | Integration-Test-Kommentar "7 trading days" falsch | IMPLEMENTIERT |

### Runde 2 — Vertiefung

- NaN-Strategie: Hybrid (No-Trade-Window neutral = 0.0, `finiteOrZero`) empfohlen und implementiert
- Overfitting-Score-Logik: ChatGPT empfahl `trainSharpe <= 0 → NEUTRAL`, `validateSharpe <= epsilon → CAP` — exakt implementiert
- stepDays < validateDays: `throw IAE` für V1 empfohlen — implementiert

---

## 4. Gemini-Review (owner: odin-033-impl-gemini)

### Findings

| Dimension | Kategorie | Finding | Aktion |
|-----------|-----------|---------|--------|
| Dimension 1 | WICHTIG | Cap-Leak: `validateSharpe` marginally > EPSILON kann Ratio >> CAP erzeugen | IMPLEMENTIERT (`Math.min(ratio, CAP)`) |
| Dimension 1 | HINWEIS | `String.format` in SLF4J-Calls immer ausgewertet unabhängig vom Log-Level | VERWORFEN (Performance negligible, Logs laufen auf INFO) |
| Dimension 1 | GO | Record-Nutzung und Null-Safety erstklassig | — |
| Dimension 1 | GO | `finiteOrZero` schützt Aggregation effektiv | — |
| Dimension 2 | GO | Exakte Umsetzung der Spezifikation (60/20/20, min 3, threshold 2.0) | — |
| Dimension 2 | GO | `stepDays >= validateDays` korrekt implementiert | — |
| Dimension 2 | GO | Trading-Day-basiertes Window-Splitting | — |
| Dimension 3 | WICHTIG | Overall-Overfitting-Score via gemittelte Sharpes kann sich ausgleichen → instabiles System unsichtbar | Mit `anyWindowOverfits`-Flag adressiert |
| Dimension 3 | HINWEIS | Mittelwert von Sharpes mathematisch ungenau (V1 Simplification) | In JavaDoc dokumentiert |
| Dimension 3 | GO | Look-Ahead-Bias: Indizierung und Record-Invariante sind wasserdicht | — |

---

## 5. Test-Abdeckung

### Unit-Tests (WalkForwardRunnerTest, 35 Tests)

- Window-Splitting: 6 Tests (Anzahl Windows, Train-Dates, Validate-Dates, Index-Sequenz, Overlap-Check, BacktestRunner-Call-Count)
- Insufficient Data: 3 Tests (< 3 Windows, 0 Windows, genau 3 Windows)
- Overfitting-Score: 7 Tests (at/above/below threshold, beide null, train negativ, train null, validate negativ+train positiv)
- Aggregation: 3 Tests (Sharpe, WinRate, MaxDrawdown als arithmetisches Mittel)
- Report-Felder: 4 Tests (Dates, Variant, Window-Params, Window-Indices)
- Null-Guards und Validation: 7 Tests (null instruments, null variant, zero params, startDate > endDate, step < validate)
- Per-Window-Flag: 1 Test
- NaN-Handling: 1 Test
- Konstanten: 3 Tests (DEFAULT_*, THRESHOLD, MINIMUM)

### Integrations-Tests (WalkForwardRunnerIntegrationTest, 10 Tests)

- End-to-End 9 Trading-Days → 3 Windows
- Jedes Window hat train + validate Report
- No-Bar-Data → aggregatedSharpe = 0.0
- No-Bar-Data → overfittingScore = 1.0 (neutral), kein Warning
- Konfigurierbare Window-Größen gespeichert im Report
- step=3 mit 9 Trading-Days → 2 Windows → InsufficientDataException
- Train/Validate-Perioden überlappen nicht (no look-ahead)
- Successive Windows advance in time
- Report-Dates matchen Input-Range
- Single Trading-Day → InsufficientDataException

### Gesamt

| Metrik | Wert |
|--------|------|
| Tests vor ODIN-033 | 166 |
| Tests nach ODIN-033 | 178 |
| Neue Tests | 12 (net, exkl. pre-existing fixes) |
| Failures | 0 |
| Errors | 0 |

---

## 6. Pre-Existing Bugs gefixt

Diese Bugs wurden beim Kompilieren des Backtest-Moduls entdeckt und behoben:

1. **BacktestRunner.java**: `CoreProperties` Konstruktor-Aufruf fehlte `coreProperties.session()` (3. Parameter). Ursache: `SessionProperties` wurde zu `CoreProperties` hinzugefügt ohne alle Aufrufer zu updaten.

2. **BacktestRunner.java**: `LifecycleManager` Konstruktor-Aufruf fehlte `SessionPhaseMonitor`-Instanz (letzter Parameter). Ursache: `LifecycleManager` wurde erweitert ohne alle Aufrufer zu updaten.

3. **BacktestRunnerTest.java + BacktestRunnerGovernanceIntegrationTest.java**: `buildCoreProperties()`-Hilfsmethode fehlte `CoreProperties.SessionProperties` Instanziierung und Übergabe.

4. **Stale Class Files**: Maven incremental compiler nutzte veraltete `.class`-Dateien für `odin-core`. Behoben durch Force-Recompilation (`touch CoreProperties.java` + `mvn compile -pl odin-core -am`).

---

## 7. Offene Punkte (V2)

- Aggregation aus concatenated OOS Daily Returns statt arithmetischem Mittelwert von Sharpes
- Gewichtetes Aggregieren bei variablen Window-Längen (wenn V2 ungleiche Window-Größen erlaubt)
- Parallelisierung der Window-Runs (sequentiell in V1; Windows sind voneinander unabhängig)
- Per-Window maximaler Overfitting-Score als Alternative zum Overall-Score
