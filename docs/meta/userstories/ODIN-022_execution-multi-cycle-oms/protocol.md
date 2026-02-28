# ODIN-022 — Multi-Cycle OMS Handling: Implementation Protocol

**Story:** ODIN-022 — Multi-Cycle OMS Handling
**Datum:** 2026-02-22
**Status:** Implementierung abgeschlossen, Reviews abgeschlossen, kein Commit (per Auftrag)

---

## 1. Implementierungsübersicht

### 1.1 Neue Dateien

| Datei | Beschreibung |
|-------|-------------|
| `odin-execution/src/main/java/de/its/odin/execution/model/CycleContext.java` | Immutable Record: cycleNumber, dailyCapital, remainingBudget mit Compact-Constructor-Validierung |
| `odin-execution/src/test/java/de/its/odin/execution/oms/OmsMultiCycleIntegrationTest.java` | 5 Integrationstests für den vollständigen Multi-Cycle-Flow (kein Spring-Kontext) |

### 1.2 Geänderte Dateien

| Datei | Änderungen |
|-------|-----------|
| `OrderManagementService.java` | `submitEntry(SizedOrder, CycleContext)`, `notifyCycleEnd()`, Felder `dayRealizedPnl`, `dayRoundTripCount`, `activeCycleContext`, erweiterte `reset()` |
| `ExecutionProperties.java` | Neues `OmsProperties`-Record: `cycle2SizeFactor`, `minBudgetThresholdPct` |
| `PositionSizer.java` | Akzeptiert `OmsProperties` im Konstruktor, wendet `cycle2SizeFactor` bei `cycleNumber > 1` an |
| `odin-execution.properties` | Neue Keys: `odin.execution.oms.cycle2-size-factor=0.7`, `odin.execution.oms.min-budget-threshold-pct=0.02` |
| `OrderManagementServiceTest.java` | 11 neue ODIN-022-Tests (9 Cycle-aware + 2 neue Guards-Tests), Import `CycleContext` hinzugefügt, Helper-Methoden für Multi-Cycle-Tests |
| `PositionSizerTest.java` | 4 neue Cycle-Sizing-Tests |

---

## 2. Architektur-Entscheidungen

### 2.1 Budget-Berechnung liegt bei odin-core

Das OMS berechnet das Budget NICHT selbst. Es akzeptiert `remainingBudget` als gegeben (kommt von odin-core, das die Formel `(dailyCapital × 10%) − |realisedLosses|` ausführt). Nur Verluste reduzieren das Budget; Gewinne erhöhen es nicht.

### 2.2 Cycle-2+ Sizing liegt bei PositionSizer (nicht im OMS)

Der `cycle2SizeFactor` wird von `PositionSizer.calculateShares(cycleNumber)` angewendet. Das OMS erhält einen bereits größenreduzierten `SizedOrder`. Eine doppelte Reduktion im OMS wäre falsch gewesen.

### 2.3 clientOrderSequence wird niemals zurückgesetzt

Monoton steigend über alle Zyklen eines Runs, um ClientOrderId-Kollisionen zu vermeiden. `reset()` setzt diese Zahl bewusst nicht zurück.

### 2.4 notifyCycleEnd() ist idempotent

Nach dem ersten erfolgreichen Aufruf wird `activeCycleContext` auf `null` gesetzt. Weitere Aufrufe treffen den Null-Guard und kehren ohne Effekt zurück. So werden keine Metriken doppelt gezählt.

---

## 3. ChatGPT Review — Round 1

**Datum:** 2026-02-22 | **Slot:** 0 | **Dateien:** 5

### Findings und Umsetzung

| Prio | Finding | Umsetzung |
|------|---------|-----------|
| KRITISCH-1 | `clientOrderSequence = 0` in `reset()` verursacht ClientOrderId-Kollisionen über Zyklen | **Behoben:** `clientOrderSequence = 0` aus `reset()` entfernt, Kommentar hinzugefügt |
| KRITISCH-2 | OMS validiert Budget-Formel (gains ≠ budget erhöhend) nicht selbst | **Dokumentiert:** OMS nimmt `remainingBudget` als gegeben; Architektur delegiert Berechnung an odin-core |
| WICHTIG-1 | Legacy `submitEntry(SizedOrder)` umgeht Budget-Gate | **Kommentiert:** Prominent-Warning in JavaDoc |
| WICHTIG-2 | `notifyCycleEnd()` nicht garantiert wenn Caller `reset()` direkt aufruft | **Behoben:** Guard-Warning in `reset()` wenn `activeCycleContext != null` |
| WICHTIG-3 | Fehlende Obergrenze bei `OmsProperties`-Feldern | **Behoben:** `@DecimalMax("1.0")` zu `cycle2SizeFactor` und `minBudgetThresholdPct` hinzugefügt |
| WICHTIG-4 | JavaDoc-Mismatch in `CycleContext` (Budget-Basis unklar) | **Behoben:** JavaDoc in `CycleContext.java` präzisiert |

---

## 4. ChatGPT Review — Round 2

**Datum:** 2026-02-22 | **Slot:** 0 (gleicher Kontext) | **Focus:** Korrektheit der R1-Fixes + neue Grenzfälle

### Findings und Umsetzung

| Prio | Finding | Umsetzung |
|------|---------|-----------|
| KRITISCH | `reset()` ohne `cancelAllOrders()` = Orphan Orders bei vorzeitigem Reset | **Teilweise:** Zusätzlicher Guard-Warning für `positionState.isPositioned()` in `reset()`. `IllegalStateException` abgelehnt — würde Recovery-Code verhindern; odin-core trägt die Lifecycle-Verantwortung |
| WICHTIG | Kein Guard gegen Doppel-Entry wenn `activeCycleContext != null` | **Behoben:** Guard in `submitEntry(SizedOrder, CycleContext)` — gibt `false` zurück wenn Zyklus noch aktiv |
| WICHTIG | `notifyCycleEnd()` nicht idempotent — `dayRoundTripCount` wurde mehrfach hochgezählt | **Behoben:** `activeCycleContext = null` am Ende von `notifyCycleEnd()` — zweite Aufrufe treffen Null-Guard |
| NICE | Thread-Safety-Kommentar unvollständig | **Behoben:** Kommentar erweitert, erklärt dass BrokerGateway Callbacks auf Pipeline-Thread serialisiert |

**Neue Tests hinzugefügt:**
- `submitEntryWithActiveCycleContextRejected` — verifiziert Doppel-Entry-Guard
- `notifyCycleEndIsIdempotent` — verifiziert dass zweiter Aufruf kein Doppelzählen verursacht

---

## 5. Gemini Review — 3 Dimensionen

**Datum:** 2026-02-22

### Dimension 1: Code Quality

| Prio | Finding | Entscheidung |
|------|---------|-------------|
| WICHTIG | Legacy-Konstruktor ohne `PositionUpdateListener` sollte `@Deprecated` sein | Abgelehnt — Konstruktor ist für Tests und backwards-compatibility absichtlich erhalten |
| NICE | `OrderManagementService` wächst zur God Class | Akzeptiert als technische Schuld; Auslagerung des Day-Session-Trackings in `DaySessionManager` ist Kandidat für spätere Iteration |

### Dimension 2: Concept Compliance

| Prio | Finding | Entscheidung |
|------|---------|-------------|
| KRITISCH (Fehlalarm) | `cycle2SizeFactor` wird im OMS nicht angewendet | **Kein Bug:** Sizing liegt bewusst bei `PositionSizer`. JavaDoc-Mismatch in `CycleContext` behoben. |
| WICHTIG | Irreführendes Namespacing: `cycle2SizeFactor` in `OmsProperties` obwohl von `PositionSizer` gelesen | Akzeptiert: Property bleibt in `OmsProperties` als konzeptioneller Single-Source-of-Truth für Multi-Cycle-Konfiguration. Beide Klassen lesen aus demselben Namespace. |

**Behobene Documentation-Drift:** `CycleContext.java` JavaDoc korrigiert — macht jetzt klar, dass das OMS das Sizing NICHT selbst anwendet, sondern `PositionSizer` die Reduktion vor der `SizedOrder`-Erstellung vornimmt.

### Dimension 3: Production Readiness

| Prio | Finding | Entscheidung |
|------|---------|-------------|
| KRITISCH | `reset()` bei offener Position löscht OMS-Gedächtnis → Orphan Orders | **Teilweise behoben:** Warning-Log; keine `IllegalStateException` (würde Cleanup verhindern). Architekturvertrag: odin-core schließt Position vor `reset()`. |
| NICE | Double-Entry Guard (bereits behoben in R2) | Behoben |
| NICE | Audit Trail `CYCLE_START/CYCLE_END` vollständig | OK |

---

## 6. Testergebnisse

### Unit Tests (`mvn test -pl odin-execution`)

```
Tests run: 184, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

**Neue Tests in dieser Story (11 + 4 + 2 = 17 neue Tests):**

**OrderManagementServiceTest.java (11 neue):**
- `submitEntryWithCycleContextLogsStartEventAndAcceptsEntry`
- `submitEntryWithCycleContextBlockedWhenBudgetBelowThreshold`
- `submitEntryWithCycleContextBlockedExactlyAtThreshold`
- `submitEntryWithCycleContextStoresCycleContextForLaterUse`
- `notifyCycleEndLogsEndEventAndAccumulatesDayMetrics`
- `notifyCycleEndWithoutActiveCycleContextLogsWarningAndReturns`
- `resetPreservesDayMetricsButClearsCycleState`
- `twoCyclesDayRealizedPnlAccumulates`
- `budgetGatePreventsEntriesOnExhaustedBudget`
- `submitEntryWithActiveCycleContextRejected` *(nach R2)*
- `notifyCycleEndIsIdempotent` *(nach R2)*

**PositionSizerTest.java (4 neue):**
- `cycle2SizingReducesSharesByConfiguredFactor`
- `cycle1SizingMatchesConvenienceOverload`
- `cycle3SizingAppliesSameFactorAsyCycle2`
- `cycle2SizingWithZeroSharesStaysZero`

### Integration Tests (`OmsMultiCycleIntegrationTest`)

```
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
(Klasse nach Failsafe-Namenskonvention *IntegrationTest — läuft unter Maven Failsafe)
```

**Tests:**
- `fullCycle1ToResetToCycle2AccumulatesDayMetrics`
- `budgetExhaustionBlocksCycleEntry`
- `dayMetricsAccumulateCorrectlyOverMultipleCycles`
- `multipleResetsPreserveDayMetrics`
- `eventLogContainsCycleStartAndEndForEachCycle`

---

## 7. Bekannte Einschränkungen (bewusste Entscheidungen)

1. **reset() bei offener Position**: Logs Warning, wirft keine Exception. Begründung: Eine Exception in `reset()` würde Recovery-Code in odin-core verhindern, falls ein Fehler-Szenario aufgetreten ist. Der Architekturvertrag legt fest, dass odin-core die Position vor `reset()` schließen muss.

2. **Legacy-Overload `submitEntry(SizedOrder)`**: Bleibt erhalten für Tests und Backtest-Szenarien ohne CycleContext. Documented mit Bypass-Warning.

3. **Budget-Berechnung nicht im OMS**: Das OMS vertraut auf `remainingBudget` aus `CycleContext`. Absichtlich so designed — OMS hat keine Kenntnis der Tageshistorie; das liegt bei odin-core.
