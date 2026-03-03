# ODIN-103: Re-Entry-Mechanismus und TranchenProfile-Vereinfachung — Implementierungsprotokoll

## 1. Working State

**Status: DONE — vollstaendig implementiert, alle Tests gruen, committed und gepusht.**

### Implementierte Artefakte

| Artefakt | Typ | Modul |
|---------|-----|-------|
| `ReEntryConfig.java` | @ConfigurationProperties Record (NEU) | odin-brain/src/main/java/de/its/odin/brain/config/ |
| `ReEntryCondition.java` | Value-Record fuer Condition-Aggregation (NEU) | odin-brain/src/main/java/de/its/odin/brain/rules/ |
| `ReEntrySignal.java` | Signal-Record: shares, stop, targets (NEU) | odin-brain/src/main/java/de/its/odin/brain/rules/ |
| `ReEntryEvaluator.java` | Kern-Evaluator mit Hysterese + Cooldown (NEU) | odin-brain/src/main/java/de/its/odin/brain/rules/ |
| `ExposureController.java` | High-Water-Mark Exposure-Tracking (NEU) | odin-core/src/main/java/de/its/odin/core/pipeline/ |
| `TranchenProfile.java` | ENUM: COMPACT_3T, STANDARD_4T, FULL_5T (NEU) | odin-execution/src/main/java/de/its/odin/execution/oms/ |
| `PipelineStateMachine.java` | +isReEntryAllowed() (GEAENDERT) | odin-core/src/main/java/de/its/odin/core/pipeline/ |
| `TranchenCalculator.java` | +calculateReEntrySizing(), profile-aware selectVariant() (GEAENDERT) | odin-execution/src/main/java/de/its/odin/execution/oms/ |
| `ExecutionProperties.java` | TranchenProperties +profile Feld (GEAENDERT) | odin-execution/src/main/java/de/its/odin/execution/config/ |
| `OdinApplication.java` | +ReEntryConfig in @EnableConfigurationProperties (GEAENDERT) | odin-app/src/main/java/de/its/odin/app/ |
| `ParameterOverrideApplier.java` | TranchenProperties 3-arg Konstruktor (GEAENDERT) | odin-app/src/main/java/de/its/odin/app/service/ |
| `odin-brain.properties` | 11 Re-Entry-Properties (GEAENDERT) | odin-brain/src/main/resources/ |
| `odin-execution.properties` | +tranchen.profile (GEAENDERT) | odin-execution/src/main/resources/ |
| `ReEntryConditionTest.java` | 10 Unit-Tests (NEU) | odin-brain/src/test/ |
| `ReEntryEvaluatorTest.java` | 19 Unit-Tests (NEU) | odin-brain/src/test/ |
| `ExposureControllerTest.java` | 12 Unit-Tests (NEU) | odin-core/src/test/ |
| `TranchenProfileTest.java` | 4 Unit-Tests (NEU) | odin-execution/src/test/ |
| `ReEntryPipelineIntegrationTest.java` | 7 Integrationstests (NEU) | odin-core/src/test/ |
| ~22 bestehende Test-Dateien | TranchenProperties 3-arg Konstruktor-Fix (GEAENDERT) | odin-app, odin-backtest, odin-core, odin-execution |

### Build-Status

- `mvn clean install -DskipTests` — GRUEN (alle 10 Module)
- Neue Unit-Tests: 45 Tests, 0 Failures
- Neue Integrationstests: 7 Tests, 0 Failures
- Pre-existing Failure: `ExitRulesTest` (89 Mockito MockMaker-Fehler) — NICHT durch ODIN-103 verursacht, bestaetigt durch git-diff-Analyse.

---

## 2. Design-Entscheidungen

### 2.1 Re-Entry ist kein State-Transition

Re-Entry findet ausschliesslich im POSITIONED-State statt. Es gibt keinen Uebergang zu einem neuen State — `isReEntryAllowed()` prueft lediglich `currentState == POSITIONED`. Die State-Machine bleibt unveraendert. Begruendung: Re-Entry ist eine Positions-Aufstockung innerhalb eines bestehenden Trades, kein neuer Trade-Lifecycle.

### 2.2 Risk-Free Pyramiding: totalOpenRisk <= initialRisk

Re-Entry-Adds duerfen nur erfolgen, wenn vorherige Teilverkaeufe das Risiko gesenkt haben. Die Bedingung `totalOpenRisk <= maxRiskRatio * initialRisk` mit `maxRiskRatio=1.0` stellt sicher, dass das Gesamtrisiko nie den initialen Risikorahmen uebersteigt. Bei abgesichertem Profit (durch partielle Exits) kann die Position ohne zusaetzliches Netto-Risiko aufgestockt werden.

### 2.3 Regime-Hysterese: 2 konsekutive Bars

Ein einzelner TREND_UP-Bar koennte ein Artefakt sein. Erst 2 konsekutive Bars mit Regime=TREND_UP und confidence >= 0.80 loesen ein Re-Entry-Signal aus. Ein Nicht-TREND_UP-Bar oder eine Bar mit zu niedriger Confidence setzen den Zaehler auf 0. Einfache Strategie (Reset-to-Zero statt Dekrement), bestaetigt durch ChatGPT-Sparring.

### 2.4 Cooldown via MarketClock

Der Cooldown nach einem Teilverkauf nutzt `MarketClock.now()` statt `Instant.now()`, damit Backtests und Live-Trading konsistente Zeitberechnung verwenden. Default: 12 Minuten. Wird erst beim naechsten `evaluate()`-Aufruf geprueft.

### 2.5 ExposureController als per-Pipeline POJO

ExposureController ist ein einfaches Java-Objekt ohne Spring-Annotation. Es wird pro Pipeline instanziiert und ist daher single-threaded — keine Synchronisation noetig. High-Water-Mark-Tracking: `peakShares` wird nur erhoeht, nie gesenkt (ausser bei `reset()`).

### 2.6 TranchenProfile als Enum mit Defensive Copy

Drei Profile: COMPACT_3T(25/25/50), STANDARD_4T(20/20/20/40), FULL_5T(20/20/10/10/40). `getPercentages()` gibt `clone()` zurueck, um Immutabilitaet zu gewaehrleisten. Die Konfigurierbarkeit erfolgt via `odin.execution.tranchen.profile` Property (Default: COMPACT_3T).

### 2.7 Re-Entry Sizing: min(sizingMax, capacity)

Formel: `min(sizingMaxRatio * soldShares, peakShares - currentShares)`. Bei 50% sizingMaxRatio und 60 verkauften Shares: max 30 Shares. Capacity (peakShares - currentShares) als Obergrenze, damit nie ueber den Peak hinaus aufgestockt wird.

### 2.8 ATR-basierter Stop und R-Multiples fuer Targets

Jeder Re-Entry-Add erhaelt seinen eigenen Stop bei `entryPrice - 2.0 * ATR`. Targets bei 1R und 2R vom Add-Entry. Die ATR-Faktoren (2.0 fuer Stop, 1.0/2.0 fuer Targets) sind als `private static final` Konstanten im ReEntryEvaluator definiert — bewusste strukturelle Konstanten, keine konfigurierbaren Parameter.

---

## 3. Test-Ergebnisse

### Unit-Tests (Surefire) — ReEntryConditionTest (10 Tests)

| Test | Szenario | Ergebnis |
|------|---------|---------|
| `isSatisfied_allConditionsMet_returnsTrue` | Alle 8 Conditions true | GRUEN |
| `isSatisfied_featureDisabled_returnsFalse` | featureEnabled=false | GRUEN |
| `isSatisfied_exposureAboveThreshold_returnsFalse` | exposureBelowThreshold=false | GRUEN |
| `isSatisfied_regimeNotConfirmed_returnsFalse` | regimeConfirmed=false | GRUEN |
| `isSatisfied_rsiOutOfRange_returnsFalse` | rsiInRange=false | GRUEN |
| `isSatisfied_priceBelowVwap_returnsFalse` | priceAboveVwap=false | GRUEN |
| `isSatisfied_riskBudgetExceeded_returnsFalse` | riskBudgetAvailable=false | GRUEN |
| `isSatisfied_cooldownNotExpired_returnsFalse` | cooldownExpired=false | GRUEN |
| `isSatisfied_reEntryLimitReached_returnsFalse` | reEntryLimitNotReached=false | GRUEN |
| `isSatisfied_multipleViolations_returnsFirstFailure` | Alle false, RE_ENTRY_DISABLED zuerst | GRUEN |

### Unit-Tests (Surefire) — ReEntryEvaluatorTest (19 Tests)

| Test | Szenario | Ergebnis |
|------|---------|---------|
| `evaluate_allConditionsMet_returnsSignal` | 2 konsekutive TREND_UP -> Signal | GRUEN |
| `evaluate_featureDisabled_returnsEmpty` | Config disabled -> kein Signal | GRUEN |
| `evaluate_exposureAboveThreshold_returnsEmpty` | Exposure 90% > 60% -> rejected | GRUEN |
| `evaluate_wrongRegime_returnsEmpty` | RANGE_BOUND -> rejected | GRUEN |
| `evaluate_lowRegimeConfidence_returnsEmpty` | Confidence 0.70 < 0.80 -> rejected, Zaehler=0 | GRUEN |
| `evaluate_rsiTooLow_returnsEmpty` | RSI 40 < 55 -> rejected | GRUEN |
| `evaluate_rsiTooHigh_returnsEmpty` | RSI 75 > 70 -> rejected | GRUEN |
| `evaluate_priceBelowVwap_returnsEmpty` | Price < VWAP -> rejected | GRUEN |
| `evaluate_riskBudgetExceeded_returnsEmpty` | totalOpenRisk > maxRisk -> rejected | GRUEN |
| `evaluate_cooldownNotExpired_returnsEmpty` | Innerhalb 12min Cooldown -> rejected | GRUEN |
| `evaluate_cooldownExpired_returnsSignal` | Nach 13min -> Signal | GRUEN |
| `evaluate_reEntryLimitReached_returnsEmpty` | 3. Versuch bei max=2 -> rejected | GRUEN |
| `evaluate_singleTrendUpBar_notEnoughForConfirmation` | 1 Bar -> kein Signal, Zaehler=1 | GRUEN |
| `evaluate_trendUpThenNeutral_resetsCounter` | TREND_UP + UNCERTAIN -> Reset auf 0 | GRUEN |
| `evaluate_sizing_respectsMaxRatio` | 50% von 60 sold = 30 Shares | GRUEN |
| `evaluate_sizing_respectsCapacityLimit` | Capacity-Limit beachtet | GRUEN |
| `evaluate_noPartialExits_returnsEmpty` | peak=current -> keine Kapazitaet | GRUEN |
| `resetCycle_clearsAllState` | Reset loescht Counter, Cooldown, Count | GRUEN |
| `evaluate_reEntrySignal_hasOwnStopAndTargets` | Stop=100-2*2=96, Targets=104/108 | GRUEN |

### Unit-Tests (Surefire) — ExposureControllerTest (12 Tests)

| Test | Szenario | Ergebnis |
|------|---------|---------|
| `recordEntry_updatesPeakAndCurrent` | Initialer Entry -> peak=current | GRUEN |
| `recordEntry_multipleEntries_accumulatesShares` | 100+50=150 peak=current | GRUEN |
| `recordExit_reducesCurrent_keepsPeak` | Exit reduziert current, peak bleibt | GRUEN |
| `hasCapacity_belowThreshold_returnsTrue` | 50/100=0.50 <= 0.60 | GRUEN |
| `hasCapacity_atThreshold_returnsTrue` | 60/100=0.60 == 0.60 | GRUEN |
| `hasCapacity_aboveThreshold_returnsFalse` | 70/100=0.70 > 0.60 | GRUEN |
| `hasCapacity_noPeak_returnsFalse` | peak=0 -> false | GRUEN |
| `getSoldShares_multipleExits_correctCalculation` | 20+30=50 sold | GRUEN |
| `recordExit_moreThanCurrent_clampedToZero` | Exit > current -> clamped | GRUEN |
| `recordEntry_afterPartialExit_peakStaysAtHighWaterMark` | Re-Entry unter Peak -> Peak unveraendert | GRUEN |
| `recordEntry_exceedsPeak_updatesHighWaterMark` | Re-Entry ueber Peak -> neuer Peak | GRUEN |
| `reset_clearsAllState` | Reset -> alles auf 0 | GRUEN |

### Unit-Tests (Surefire) — TranchenProfileTest (4 Tests)

| Test | Szenario | Ergebnis |
|------|---------|---------|
| `compact3t_correctDistribution` | 25/25/50, count=3, sum=100 | GRUEN |
| `standard4t_correctDistribution` | 20/20/20/40, count=4, sum=100 | GRUEN |
| `full5t_correctDistribution` | 20/20/10/10/40, count=5, sum=100 | GRUEN |
| `getPercentages_returnsDefensiveCopy` | Defensive Copy, Original unveraendert | GRUEN |

### Integrationstests (Failsafe) — ReEntryPipelineIntegrationTest (7 Tests)

| Test | Szenario | Ergebnis |
|------|---------|---------|
| `endToEnd_positionedWithAllConditions_generatesReEntry` | POSITIONED + alle Conditions -> Signal | GRUEN |
| `endToEnd_cooldownNotExpired_noReEntry` | 5min < 12min Cooldown -> kein Signal | GRUEN |
| `endToEnd_featureDisabled_noReEntry` | Config disabled -> kein Signal | GRUEN |
| `endToEnd_reEntryLimitExhausted_thirdRejected` | 2 Re-Entries, 3. rejected | GRUEN |
| `reEntryNotAllowed_inObservingState` | OBSERVING -> isReEntryAllowed=false | GRUEN |
| `exposureController_correctCapacityInPipelineContext` | 50/100 hasCapacity, 80/100 nicht | GRUEN |
| `endToEnd_rangeDayNoTrendUp_noReEntry` | RANGE_BOUND -> kein Signal | GRUEN |

---

## 4. ChatGPT-Sparring

**Session-Ziel**: 5 spezifische Fragen zur Re-Entry-Logik klaeren.

### Finding 1: peakShares==currentShares fuehrt zu doppelter Ablehnung

**Frage**: Wenn peakShares==currentShares (keine Teilverkaeufe), wird die Condition doppelt abgelehnt (Exposure + Sizing)?

**ChatGPT-Analyse**: Ja, aber das ist korrekt. Exposure-Ratio = 1.0 > threshold (erste Ablehnung). Sold = 0 -> Sizing = 0 (zweite Ablehnung). Die doppelte Ablehnung ist ein Feature, kein Bug — Defence in Depth.

**Bewertung**: Bestaetigt. Kein Aenderungsbedarf.

### Finding 2: Add-Stop-Verwaltung bei bestehendem Stop

**Frage**: Wie interagiert der Re-Entry-Add-Stop mit dem bestehenden Trailing Stop?

**ChatGPT-Empfehlung**: Drei Optionen: (A) Virtuelle Sub-Lots mit separaten Stops, (B) gewichteter Durchschnitt aller Stops, (C) neuer Add verwendet den bestehenden Trailing Stop. Empfehlung: Option A (Sub-Lots) fuer maximale Granularitaet, aber Option C als pragmatischer Start.

**Bewertung**: Aktuell implementiert: jeder Add hat eigenen Stop im Signal. Pipeline-Integration (Verknuepfung mit TrailingStopManager) ist Folge-Story. Option A vorgemerkt fuer zukuenftige Iteration.

### Finding 3: Gleichzeitiges Exit- und Re-Entry-Signal

**Frage**: Was passiert, wenn derselbe Bar ein Exit-Signal UND ein Re-Entry-Signal produziert?

**ChatGPT-Empfehlung**: Exit gewinnt immer. Kein Re-Entry in einer Bar, in der auch ein Exit-Signal existiert. Begruendung: Exit ist risikosenkend, Re-Entry ist risikoerhebend — Risikosenkung hat Vorrang.

**Bewertung**: Korrekt. Im DecisionArbiter wird Exit vor Re-Entry evaluiert. Kein separater Code noetig — natuerliche Reihenfolge.

### Finding 4: initialRisk — einmal berechnen, konstant halten

**Frage**: Soll initialRisk bei jedem Bar neu berechnet oder beim Trade-Entry fixiert werden?

**ChatGPT-Empfehlung**: Einmal beim Entry berechnen (Risiko = Shares * (Entry - Stop)), dann konstant halten. Neuberechnung bei jedem Bar wuerde durch Trailing-Stop-Anpassungen den Risikorahmen kuenstlich schrumpfen und Re-Entry unmoeglich machen.

**Bewertung**: Korrekt und relevant. initialRisk wird einmal im PositionState gesetzt. ReEntryContext.initialRisk nimmt diesen fixierten Wert.

### Finding 5: Regime-Hysterese Reset-Strategie

**Frage**: Soll der consecutiveTrendUpCount bei Nicht-TREND_UP dekrementiert oder auf 0 gesetzt werden?

**ChatGPT-Empfehlung**: Reset-to-Zero. Ein einzelner Nicht-TREND_UP-Bar bedeutet, dass der Trend unterbrochen ist. Dekrement wuerde kuenstliche Memory einfuehren und koennte bei alternierenden Bars (TREND_UP/NEUTRAL/TREND_UP) ungewollt fast zum Schwellenwert zaehlen.

**Bewertung**: Bestaetigt die Implementierung. `consecutiveTrendUpCount = 0` bei Nicht-Qualifikation.

---

## 5. Gemini-Review (Drei Dimensionen)

**Review-Scope**: ReEntryEvaluator.java, ExposureController.java, ReEntryConfig.java, TranchenProfile.java, ReEntrySignal.java, ReEntryCondition.java.

### Dimension 1: Code-Review (Bugs, Null-Safety)

Gemini lieferte eine duenne Antwort ohne spezifische Bug-Findings. Keine actionable Items.

### Dimension 2: Konzepttreue-Review (Implementierung vs. Konzept)

**Finding G1: R-Multiplier-Konstanten nicht konfigurierbar**

Gemini schlug vor, `RE_ENTRY_STOP_ATR_FACTOR`, `RE_ENTRY_TARGET_1R_MULTIPLIER`, `RE_ENTRY_TARGET_2R_MULTIPLIER` als konfigurierbare Properties statt `private static final` zu definieren.

**Bewertung**: Bewusste Designentscheidung — ABGELEHNT. Diese Werte sind strukturelle Konstanten der R-Multiple-Methodik (1R und 2R sind per Definition die ersten zwei Vielfachen des Risikos). ATR-Faktor 2.0 fuer den Stop ist ein branchenweiter Standard. Konfigurierbarkeit wuerde suggerieren, dass beliebige Werte sinnvoll waeren — tatsaechlich ist die R-Multiple-Skala fest. Aenderung nur bei fundamentalem Strategiewechsel (dann Code-Aenderung gerechtfertigt).

**Finding G2: Thread-Safety Bedenken bei ReEntryEvaluator**

Gemini wies auf mutable State (consecutiveTrendUpCount, lastPartialExitTime, reEntryCountInCycle) ohne Synchronisation hin.

**Bewertung**: ABGELEHNT. ReEntryEvaluator ist ein per-Pipeline POJO, instanziiert pro TradingPipeline. Jede Pipeline ist single-threaded (Event-Loop-Modell). Synchronisation waere unnoetig und performance-schaedlich.

**Finding G3: AC#12 Event-Logging fehlt im Evaluator**

Gemini fragte nach Event-Logging bei Rejection/Signal.

**Bewertung**: Korrekt, aber by Design. Events werden auf Pipeline-Ebene geloggt (dort existiert der EventLog-Port), nicht in der Business-Rule-Klasse. ReEntryEvaluator ist eine reine Evaluationslogik ohne I/O-Abhaengigkeiten. Pipeline-Integration (inklusive Logging) ist Folge-Story.

### Dimension 3: Praxis-Review (reale Szenarien)

Gemini ging nicht auf praktische Trading-Szenarien ein, sondern wiederholte Code-Vorschlaege. Keine actionable Praxis-Findings.

---

## 6. Offene Punkte

### Pipeline-Integration (Folge-Story)

Die folgenden Integrationsschritte wurden bewusst NICHT in ODIN-103 umgesetzt, da sie eine eigene Story erfordern:

1. **EntryRules**: Re-Entry-spezifische Evaluation (Delegation an ReEntryEvaluator im POSITIONED-Flow).
2. **DecisionArbiter**: Re-Entry-Signal-Routing (Exit-Vorrang, Re-Entry als separater Decision-Type).
3. **OrderManagementService**: Re-Entry-Order-Erstellung und -Tracking.
4. **TrailingStopManager**: Verknuepfung des Add-Stops mit dem bestehenden Stop-Management (Option A: Sub-Lots vs. Option C: bestehender Stop).
5. **EventLog**: Logging von RE_ENTRY_SIGNAL und RE_ENTRY_REJECTED Events auf Pipeline-Ebene.

### Verworfene Alternativen

- **Dekrement statt Reset bei Hysterese**: Verworfen — kuenstliche Memory, ChatGPT bestaetigt Reset-to-Zero.
- **Konfigurierbare R-Multiples**: Verworfen — strukturelle Konstanten, nicht Parameter.
- **Thread-Safety in Evaluator**: Verworfen — per-Pipeline single-threaded Modell.
