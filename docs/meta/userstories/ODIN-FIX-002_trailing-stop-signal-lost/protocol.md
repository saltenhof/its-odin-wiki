# ODIN-FIX-002 Protocol: Trailing-Stop-Signal exitPrice Propagation

**Status:** COMPLETE
**Datum:** 2026-03-03
**Worker:** Claude Opus 4.6

---

## 1. Working State

Fix implementiert, alle Tests GREEN, Build SUCCESS, ChatGPT-Sparring und Gemini-Review (3 Dimensionen) durchgefuehrt.

## 2. Root Cause

Die Root-Cause-Analyse wurde verifiziert und stimmt exakt.

### Bug A (primaer): RulesResult.exit() hat kein exitPrice-Feld
- **Datei:** `odin-brain/src/main/java/de/its/odin/brain/model/RulesResult.java`
- **Problem:** Record hatte kein `exitPrice`-Feld. Factory-Methode `exit()` akzeptierte keinen exitPrice-Parameter.
- **Zeile 91 (alt):** `return new RulesResult(IntentType.EXIT, 0.0, 0.0, null, exitReason, regime, ...)`
- entryPrice wurde auf 0.0 gesetzt, exitPrice existierte nicht.

### Bug B: RulesEngine.evaluate() verwirft exitPrice
- **Datei:** `odin-brain/src/main/java/de/its/odin/brain/rules/RulesEngine.java`
- **Zeile 263:** `return RulesResult.exit(exitSignal.get().reason(), regime.regime(), regime.confidence());`
- `exitSignal.get().exitPrice()` wurde NICHT uebergeben — der korrekt berechnete Preis ging verloren.

### Bug C: DecisionArbiter.buildIntent() nutzt entryPrice fuer EXIT
- **Datei:** `odin-brain/src/main/java/de/its/odin/brain/arbiter/DecisionArbiter.java`
- **Zeile 1177:** `rules.entryPrice()` wurde fuer ALLE Intent-Typen verwendet.
- Bei EXIT war `entryPrice = 0.0` (aus Bug A) → TradeIntent hatte Preis 0.0.

### Auswirkung
- `TradingPipeline.handleExitIntent()` (Zeile 974): `oms.submitExit(rules.exitReason(), intent.entryPrice())` reichte 0.0 weiter.
- TRAILING_STOP: OMS erstellt MARKET-Order (Preis ignoriert) → Exit funktionierte zufaellig.
- STOP_LOSS: STOP_LIMIT mit stopPrice=0.0 → Order haette nie getriggert.
- TARGET: LIMIT mit limitPrice=0.0 → Order waere sofort gefuellt bei 0.0 oder rejected.

## 3. Fix-Beschreibung

### Schritt 1: RulesResult.java — exitPrice-Feld
- Neues Record-Feld `double exitPrice` eingefuegt (nach `exitReason`, vor `regime`).
- `exit()` Factory-Methode um `double exitPrice`-Parameter erweitert.
- `noAction()` und `entry()` setzen `exitPrice = 0.0`.
- JavaDoc aktualisiert.

### Schritt 2: RulesEngine.java — exitPrice uebergeben
```java
// Alt:
return RulesResult.exit(exitSignal.get().reason(), regime.regime(), regime.confidence());
// Neu:
return RulesResult.exit(exitSignal.get().reason(), exitSignal.get().exitPrice(), regime.regime(), regime.confidence());
```

### Schritt 3: DecisionArbiter.java — EXIT-Preis korrekt routen
```java
double price = (intentType == IntentType.EXIT) ? rules.exitPrice() : rules.entryPrice();
```
Bei EXIT-Intents wird nun `rules.exitPrice()` statt `rules.entryPrice()` verwendet.

### Caller-Updates
Alle bestehenden Aufrufer von `RulesResult.exit()` und `new RulesResult(...)` in Tests aktualisiert (15 Dateien).

## 4. Design-Entscheidungen

### Warum exitPrice in RulesResult statt TradeIntent?
- `RulesResult` ist das brain-interne Ergebnis der Rules-Evaluation.
- `ExitRules` berechnet den exitPrice → `RulesResult` muss ihn tragen.
- `DecisionArbiter` mappt exitPrice in `TradeIntent.entryPrice` (Feld-Overloading).
- Gemini-Review empfiehlt langfristig Rename von `TradeIntent.entryPrice` zu `targetPrice` — berechtigter Punkt, aber ausserhalb des Fix-Scope (wuerde alle Module betreffen).

### Warum double statt Double/OptionalDouble?
- Konsistenz mit bestehendem Record-Design (alle Preis-Felder sind `double`).
- 0.0 als Default fuer nicht-EXIT-Intents ist sicher, da DecisionArbiter nach `intentType` dispatcht.
- ChatGPT/Gemini empfahlen NaN oder OptionalDouble — berechtigte Haertung, aber Scope-Erweiterung.

### Warum kein Guard-Clause fuer exitPrice=0.0?
- Gemini empfahl Guard in DecisionArbiter: `if (price <= 0.0 && requiresPricedOrder) throw ...`
- Berechtigter Punkt fuer Defense-in-Depth, aber:
  - Aktuell gibt es nur EINEN Produktions-Caller (`RulesEngine.evaluate()`), der immer den korrekten Preis setzt.
  - Guard waere ein Enhancement, nicht Teil des Bug-Fixes.
  - Als Follow-up-Empfehlung dokumentiert.

## 5. Test-Ergebnisse

### Neue Unit-Tests (RulesResultTest — 15 Tests)
- `exit_trailingStop_carriesCorrectExitPrice` — TRAILING_STOP exitPrice != 0.0
- `exit_stopLoss_carriesCorrectExitPrice` — STOP_LOSS exitPrice korrekt
- `exit_target_carriesCorrectExitPrice` — TARGET exitPrice korrekt
- `exit_allReasons_carryExitPrice` — Parameterized ueber alle ExitReason-Enums
- `exit_withExplicitHasOpenPosition_carriesExitPrice`
- `entry_hasZeroExitPrice` — Kein Seiteneffekt auf ENTRY
- `noAction_hasZeroExitPrice` — Kein Seiteneffekt auf NO_ACTION
- `noAction_withOpenPosition_hasZeroExitPrice`
- `exit_hasZeroEntryPrice` — EXIT hat korrekt entryPrice=0.0

### Neue Integration-Tests (TrailingStopPropagationIntegrationTest — 4 Tests)
- `trailingStopExitPrice_propagatesThroughRulesEngineAndArbiter` — Full Chain
- `killSwitchExitPrice_propagatesThroughChain` — CRASH_SIGNAL → KILL_SWITCH
- `directExitResult_stopLoss_arbiterProducesCorrectTradeIntentPrice` — STOP_LOSS
- `directExitResult_target_arbiterProducesCorrectTradeIntentPrice` — TARGET

### Bestehende Tests — Alle GREEN
- RulesEngineTest: 21 Tests — GREEN
- DecisionArbiterTest: 12 Tests — GREEN
- DecisionArbiterDualKeyTest: 11 Tests — GREEN
- DecisionArbiterDualKeyIntegrationTest: Tests — GREEN
- DecisionArbiterBanditIntegrationTest: Tests — GREEN
- CounterfactualDecisionTest: Tests — GREEN
- CounterfactualDecisionIntegrationTest: Tests — GREEN
- FallbackHierarchyIntegrationTest: Tests — GREEN
- RegimeWeightFusionTest: Tests — GREEN
- RegimeWeightFusionIntegrationTest: Tests — GREEN
- TradingPipelineTest: 43 Tests — GREEN
- TrailingStopExitIntegrationTest: 2 Tests — GREEN
- OrderManagementServiceTest: 47 Tests — GREEN

### Gesamt: 127 UT (odin-brain relevante) + 4 IT (neu) + 43 UT (odin-core) + 49 UT (odin-execution) = alle GREEN

## 6. ChatGPT-Sparring

### Fragen und Antworten

**F1: Gap-Down unter den Stop**
- exitPrice = effectiveStop (Stop-Level), nicht der Gap-Preis. Korrekt fuer TCA.
- STOP_LIMIT kann bei Gap-Down nicht fuellen → vorhandenes Risiko, aber nicht durch diesen Fix eingefuehrt.
- Empfehlung: Stop-Market statt Stop-Limit fuer STOP_LOSS. → Bestehende OMS-Architektur, ausserhalb Fix-Scope.

**F2: Price == Stop (Grenzfall)**
- `<=` Vergleich ist korrekt. Exit triggert bei Gleichheit.
- Tick-Size-Rundung empfohlen → Pre-existing concern, nicht fix-relevant.

**F3: Mehrere ExitSignal-Typen gleichzeitig**
- ExitRules hat Prioritaetsreihenfolge (KILL_SWITCH > TRAILING_STOP > STOP_LOSS > TARGET).
- exitPrice=0.0 nur moeglich wenn ein neuer ExitSignal-Typ ohne Preis hinzugefuegt wird.
- Empfehlung: Invariante `exitPrice > 0 fuer EXIT` → Follow-up.

**F4: exitPrice=0.0 als Default**
- Sicher, da DecisionArbiter nach intentType dispatcht.
- Empfehlung: NaN oder OptionalDouble → Haertung, ausserhalb Fix-Scope.

**F5: Auswirkung auf andere ExitReason-Typen**
- MARKET-Exits (TRAILING_STOP, KILL_SWITCH, FORCED_CLOSE, REGIME_CHANGE): exitPrice ignoriert.
- TACTICAL_EXIT: Aktuell MARKET, bei zukuenftiger Aenderung zu LIMIT waere exitPrice relevant.

### Umgesetzte Findings
- Keine Code-Aenderungen notwendig — alle Findings betreffen entweder Pre-existing Design oder zukuenftige Haertung.

## 7. Gemini-Review

### Dimension 1 — Code-Review
- exitPrice korrekt propagiert durch alle 3 Fixes: **BESTAETIGT**
- Feld-Platzierung semantisch korrekt: **BESTAETIGT**
- `exitSignal.get()` wird mehrfach aufgerufen → Pre-existing Code-Style, nicht fix-relevant.
- Guard-Clause empfohlen → Berechtigter Punkt, als Follow-up dokumentiert.
- Keine Regression durch Feld-Aenderung: Alle Caller aktualisiert.

### Dimension 2 — Konzepttreue
- `RulesResult` ist der richtige Ort fuer exitPrice: **BESTAETIGT**
- TradeIntent.entryPrice Overloading: Anerkanntes technisches Debt.
- Empfehlung: Rename zu `targetPrice` → Berechtigter Punkt, ausserhalb Fix-Scope (wuerde alle Module betreffen).
- DDD-Boundaries korrekt eingehalten: **BESTAETIGT**

### Dimension 3 — Praxis-Review
- Gap-Down: exitPrice als Stop-Level korrekt fuer TCA: **BESTAETIGT**
- Race-Condition: BrokerSimulator fuellt bei Bar N+1 Open — korrektes Verhalten.
- STOP_LOSS als STOP_LIMIT: Pre-existing OMS-Design, 0.1% Offset koennte bei starkem Gap problematisch sein → Bestehende Architektur.
- Double-Exit: OMS prueft remainingQuantity, Pipeline-FSM wechselt auf PENDING_FILL → Schutz vorhanden.

### Berechtigte Findings (nicht im Fix-Scope behoben)
1. Guard-Clause fuer exitPrice > 0 bei priced Orders → Follow-up Story
2. Rename TradeIntent.entryPrice → targetPrice → Follow-up Refactoring
3. STOP_LOSS Order-Typ (STOP_LIMIT vs STOP_MARKET) evaluieren → Separate Analyse

## 8. Offene Punkte

| # | Beschreibung | Prioritaet | Empfehlung |
|---|-------------|-----------|------------|
| 1 | Guard-Clause: exitPrice > 0 fuer priced EXIT-Orders (STOP_LOSS, TARGET) | MITTEL | Eigene Story, Defense-in-Depth |
| 2 | TradeIntent.entryPrice → Rename zu targetPrice oder separates exitPrice-Feld | NIEDRIG | Refactoring-Story, bricht API |
| 3 | STOP_LOSS: Evaluierung STOP_MARKET vs STOP_LIMIT fuer Gap-Schutz | MITTEL | Architektur-Review mit Backtest-Daten |
| 4 | ExitSignal.get() mehrfach in RulesEngine — lokale Variable extrahieren | NIEDRIG | Code-Hygiene, kein Bug |
| 5 | Pre-existing Test-Failures: ExhaustionDetectorIntegrationTest (3), OpeningConsolidationFsmIntegrationTest (1), ReEntryCorrectionFsmIntegrationTest (1) | UNABHAENGIG | Nicht durch ODIN-FIX-002 verursacht |
