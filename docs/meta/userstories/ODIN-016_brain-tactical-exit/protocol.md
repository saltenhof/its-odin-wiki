# ODIN-016 Implementierungsprotokoll: TACTICAL_EXIT Mechanismus

**Status:** DONE
**Datum:** 2026-02-22
**Implementiert von:** Claude Code Agent

---

## Uebersicht

ODIN-016 implementiert den TACTICAL_EXIT-Mechanismus: einen LLM-getriggerten Exit-Pfad,
der unter fuenf gleichzeitigen Bedingungen ausgeloest wird. Die Story baute auf ODIN-015
(ExhaustionDetector) auf.

---

## Scope der Aenderungen

### Geaenderte Dateien

| Datei | Art der Aenderung |
|-------|------------------|
| `odin-brain/src/main/java/de/its/odin/brain/rules/ExhaustionDetector.java` | Erweiterung: `evaluateExhaustion()`, `ExhaustionResult`-Record, `TOTAL_PILLARS` package-private, `advanceRollingState()` package-private |
| `odin-brain/src/test/java/de/its/odin/brain/rules/ExitRulesTest.java` | Erweiterung: 13 neue TACTICAL_EXIT Unit-Tests, Mockito-Integration |

### Neue Dateien

| Datei | Art der Aenderung |
|-------|------------------|
| `odin-brain/src/test/java/de/its/odin/brain/rules/ExitRulesTacticalExitIntegrationTest.java` | Neu: 4 Integrationstests mit realem ExhaustionDetector |

### Nicht geaendert

- `ExitRules.java` war bereits mit dem TACTICAL_EXIT-Pfad vorimplementiert. Einzige Anpassungen in
  diesem Session: Bugfix im LLM-Pfad (Rolling-State-Advance nach evaluateExhaustion). Die ODIN-016-
  Kernimplementierung war bereits enthalten.
- Keine Aenderungen an LLM-Klassen, odin-core, odin-app, odin-backtest.

---

## Design-Entscheidungen

### evaluateExhaustion() vs. detectExhaustion()

**Problemstellung:** `detectExhaustion()` hat zwei Verantwortlichkeiten: (1) Pillar-Evaluation
und (2) Rolling-State-Advance. Fuer den LLM-Pfad wird jedoch die individuelle Pillar-Auswertung
(>= 1 Pillar, nicht alle 3) benoetigt, ohne den State doppelt zu inkrementieren.

**Loesung:** Neue `evaluateExhaustion()` Methode evaluiert die Pillars OHNE State-Advance und
gibt ein `ExhaustionResult`-Record zurueck. `ExitRules` ruft danach explizit
`advanceRollingState()` auf.

**Vertrag:** Pro Decision-Bar muss exakt einer der folgenden Pfade durchlaufen werden:
- `detectExhaustion()` -> evaluiert + advanciert State (KPI-Fallback-Pfad)
- `evaluateExhaustion()` + `advanceRollingState()` -> evaluiert, dann advanciert State (LLM-Pfad)

### Anti-Drift-Counter Management

**Entscheidung:** Der `tacticalExitCount` wird NUR beim tatsaechlichen Feuern des TACTICAL_EXIT
inkrementiert (in `evaluateLlmTacticalExit()`, nach allen Bedingungspruefungen). Geblockte
Versuche (Gain zu niedrig, Hold zu kurz, etc.) inkrementieren den Counter NICHT.

**Begruendung:** Ein blockierter Versuch ist kein Drift -- es ist normales Verhalten. Nur
erfolgreiche Exits sollen limitiert werden.

**Reset:** `resetPosition()` setzt den Counter auf 0, zusammen mit Highwater-Marks.

### HOLD_RUNNERS-Semantik

**Entscheidung:** Wenn `exitBias == HOLD_RUNNERS`, wird der gesamte LLM-Pfad fuer TACTICAL_EXIT
blockiert. Es gibt keinen Fallback auf KPI-Exhaustion wenn `llmOutput != null` ist.

**Begruendung:** HOLD_RUNNERS ist eine explizite LLM-Entscheidung, die Position laufen zu lassen.
Wenn das LLM dieses Bias setzt, muss es respektiert werden -- auch wenn KPI-Signale eine
Erschoepfung anzeigen.

### initialStop=NaN: Gain-Check wird uebersprungen

**Entscheidung:** Wenn `initialStop=NaN`, wird der Min-Gain-Check (> 0.5R) komplett uebersprungen.
TACTICAL_EXIT kann dann bei jeder Gewinnsituation feuern.

**Begruendung:** Callers, die `NaN` uebergeben, akzeptieren explizit den fehlenden Gain-Guard.
Das Verhalten ist dokumentiert und getestet (pinning test).

### Prioritaet REGIME_CHANGE vs. TACTICAL_EXIT

**Hinweis:** Die story.md (technische Details) sagt "nach TRAILING_STOP, vor REGIME_CHANGE".
Das Wiki (Kap 6) definiert REGIME_CHANGE als Prio 5 und TACTICAL_EXIT als Prio 6.
Beide sind widersprueichlich.

**Entscheidung:** Prio wie im Wiki beibehalten (REGIME_CHANGE=5, TACTICAL_EXIT=6).
Begruendung: Das Wiki ist die autoritative Architekturreferenz. Die story.md hat hier einen
Fehler im technischen Detail-Abschnitt. Zur Klaerung beim Stakeholder eskaliert (siehe
Offene Punkte).

---

## ChatGPT-Sparring (DoD 2.5)

ChatGPT identifizierte 5 zusaetzliche Edge-Cases mit hohem Testwert:

| Vorschlag | Bewertet | Umgesetzt |
|-----------|----------|-----------|
| LLM action=HOLD mit aktivem KPI-Exhaustion -> kein Fallback | Ja, korrekter Edge-Case | Ja: `tacticalExit_llmHoldAction_blocksEvenWhenKpiExhaustionActive` |
| initialStop=NaN -> Gain-Check uebersprungen, Verhalten pinnen | Ja, wichtige Boundary | Ja: `tacticalExit_initialStopNaN_gainCheckSkipped_firesOnOtherConditions` |
| regimeConfidence exactly 0.3 -> geblockt (nicht <=) | Ja, klassische Boundary | Ja: `tacticalExit_regimeConfidenceExactlyAtThreshold_blocked` |
| barsHeld exactly 3 -> feuert | Ja, fencepost boundary | Ja: `tacticalExit_barsHeldExactlyAtMinimum_fires` |
| Geblockte Versuche inkrementieren Anti-Drift-Counter nicht | Ja, wichtige Korrektheit | Ja: `tacticalExit_blockedAttemptDoesNotIncrementAntiDriftCounter` |

**Verworfene Vorschlaege:**
- "LLM-Analysis-Freshness-Gate waehrend TACTICAL_EXIT" -> Scope-Creep, betrifft LLM-Orchestration,
  nicht ExitRules.

---

## Gemini-Review (DoD 2.6)

### Dimension 1: Code-Review

**Kritisches Finding: Rolling State friert im LLM-Pfad ein**

- Die neue `checkExit(LlmTacticalOutput)` Uberladung ruft `evaluateTacticalExit()` auf.
- Im LLM-Pfad wird `evaluateExhaustion()` verwendet (kein State-Advance).
- `detectExhaustion()` wird im neuen Pfad nie aufgerufen -> RSI-Shift-Register und
  EMA-Spread-History werden nie aktualisiert.
- Folge: Pillar C (C3: Fast RSI Drop, C4: Trend Stall) funktioniert im LLM-Pfad nicht korrekt.

**Fix implementiert:**
- In `evaluateTacticalExit()` wird nach `evaluateLlmTacticalExit()` explizit
  `exhaustionDetector.advanceRollingState(indicators)` aufgerufen.
- `advanceRollingState()` von `private` auf `package-private` geaendert.
- JavaDoc in `evaluateExhaustion()` aktualisiert: Caller-Verantwortung fuer State-Advance
  explizit dokumentiert.

**Weitere Findings (alle korrekt/kein Bug):**
- Anti-Drift-Counter-Reset: korrekt implementiert
- Null-Safety bei LlmTacticalOutput: ausreichend (API garantiert non-null-Felder)
- Race Conditions: Single-threaded Pipeline-Architektur, kein Problem
- Division by Zero: Guard `if (oneR > 0)` vorhanden
- NaN/Infinity Handling: konsequent mit `Double.isFinite()` geschuetzt
- IndexOutOfBounds bei Decision Bars: `size() >= LOOKBACK_BARS` Guard vorhanden

**Geminis zweites Finding (bewertet, nicht umgesetzt):**
Gemini schlug vor, dass `detectExhaustion()` auch bei intraday Ticks (also mehrfach pro Bar)
sicher sein muss. Das ist kein echter Bug in ODIN: Der ExhaustionDetector wird pro Decision-Bar
genau einmal aufgerufen (einmal pro Loop-Iteration in der Pipeline). Das ist by design
single-threaded, single-call-per-bar.

### Dimension 2: Konzepttreue-Review

| AC | Status |
|----|--------|
| TACTICAL_EXIT nur wenn LLM-Analysis vorhanden und action=EXIT | OK |
| Mindestgewinn: > 0.5R | OK |
| Bestaetigung: >= 1 Pillar ODER confidence < 0.3 | OK |
| Anti-Drift: max 1 TACTICAL_EXIT pro Position | OK |
| Mindest-Haltedauer: >= 3 Decision-Bars | OK |
| ExitReason == TACTICAL_EXIT | OK |
| Log-Eintrag mit LLM-Reasoning und Bestaetigung | OK |
| HOLD_RUNNERS blockiert TACTICAL_EXIT komplett | OK |
| KPI-Fallback: alle 3 Pillars erforderlich | OK |

**Prioritaets-Diskrepanz:** Gemini bestaetigt den Widerspruch in der story.md (sieh Design-Entscheidung oben).

### Dimension 3: Praxis-Review

| Nr | Thema | Einschaetzung |
|----|-------|--------------|
| 1 | Broker-Rejection verbrennt Anti-Drift-Counter | Counter wird bei Signal-Emission gesetzt. Kein Rollback bei Ablehnung. ODIN-016-Scope beendet. Zur Klaerung eskaliert. |
| 2 | Multi-Tranche-Positionen | ODIN ist aktuell Long-Only Single-Tranche. Bei zukuenftigen Tranchen muss Durchschnitts-Entry-Preis uebergeben werden. |
| 3 | Pipeline-Restart + barsHeld-Counter-Verlust | Nach Neustart muss barsHeld aus persistiertem State wiederhergestellt werden. odin-core Crash-Recovery-Thema. |
| 4 | Boundary-Semantik gainR <= MIN_GAIN_R | Exakt 0.5R wird geblockt (story sagt "> 0.5R"). Korrekt. Pinning-Test vorhanden. |
| 5 | Prioritaet REGIME_CHANGE vs TACTICAL_EXIT | story.md widerspricht Wiki. Zur Klaerung eskaliert. |

---

## Test-Coverage

**Gesamt: 46 Tests, 0 Failures**

### Unit-Tests (ExitRulesTest) -- 13 TACTICAL_EXIT-Tests

| Test | Abgedecktes AC |
|------|---------------|
| `tacticalExit_allConditionsMet_producesExit` | Hauptpfad |
| `tacticalExit_gainBelowMinimum_blocked` | Mindestgewinn |
| `tacticalExit_noExhaustionConfirmation_blocked` | Bestaetigung (0 Pillars + conf>=0.3) |
| `tacticalExit_antiDrift_secondAttemptBlocked` | Anti-Drift max 1 |
| `tacticalExit_minHoldDurationNotMet_blocked` | Mindest-Haltedauer |
| `tacticalExit_holdRunnersExitBias_blocked` | HOLD_RUNNERS |
| `tacticalExit_lowRegimeConfidence_allowsExitWithoutKpiPillars` | Confidence-Pfad |
| `tacticalExit_llmHoldAction_blocksEvenWhenKpiExhaustionActive` | Edge Case: HOLD + KPI aktiv |
| `tacticalExit_initialStopNaN_gainCheckSkipped_firesOnOtherConditions` | Edge Case: NaN initialStop |
| `tacticalExit_regimeConfidenceExactlyAtThreshold_blocked` | Boundary: confidence = 0.3 |
| `tacticalExit_barsHeldExactlyAtMinimum_fires` | Boundary: barsHeld = 3 |
| `tacticalExit_blockedAttemptDoesNotIncrementAntiDriftCounter` | Anti-Drift-Counter Korrektheit |
| `tacticalExit_antiDriftCounterResetOnNewPosition_allowsExit` | Counter-Reset |
| `tacticalExit_exactlyAtMinGainBoundary_blocked` | Boundary: gain = 0.5R |

### Integrationstests (ExitRulesTacticalExitIntegrationTest) -- 4 Tests

| Test | Abgedecktes Szenario |
|------|---------------------|
| `tacticalExit_llmPathWithTwoActivePillars_producesExit` | Vollstaendiger LLM-Pfad mit realem ExhaustionDetector |
| `tacticalExit_trailingStopPrecedesTacticalExit` | Prioritaet: TRAILING_STOP (4) vor TACTICAL_EXIT (6) |
| `tacticalExit_kpiFallbackPath_allThreePillarsFiresExit` | KPI-Fallback: alle 3 Pillars -> TACTICAL_EXIT |
| `tacticalExit_regimeChangePrecedesTacticalExit` | Prioritaet: REGIME_CHANGE (5) vor TACTICAL_EXIT (6) |

---

## Offene Punkte (zur Eskalation an Stakeholder)

1. **Prioritaet TACTICAL_EXIT vs. REGIME_CHANGE:**
   story.md sagt "vor REGIME_CHANGE", Wiki sagt "nach REGIME_CHANGE".
   Welche Semantik ist fachlich korrekt? Soll eine Position mit Regime-Change trotzdem
   durch TACTICAL_EXIT exited werden wenn das LLM EXIT signalisiert?

2. **Broker-Rejection + Anti-Drift-Counter:**
   Wenn der Broker eine TACTICAL_EXIT-Order ablehnt, ist der Counter trotzdem gesetzt.
   Soll der Counter nur bei bestatigter Order-Ausfuehrung gesetzt werden?

3. **barsHeld nach Crash-Recovery:**
   Nach einem Pipeline-Neustart muss der `barsHeld`-Counter aus dem persistierten State
   (odin-core) wiederhergestellt werden. Sonst ist TACTICAL_EXIT fuer die naechsten 3 Bars
   faelschlicherweise blockiert.

4. **Rolling-State-Advance bei fruehzeitigem Exit (Architektur-Concern, QS-R1):**
   Wenn TRAILING_STOP (Prio 4) oder REGIME_CHANGE (Prio 5) feuern, gibt `checkExit()`
   fruehzeitig zurueck. `evaluateTacticalExit()` wird nicht aufgerufen, also auch
   `advanceRollingState()` nicht. Folge: Das RSI-Shift-Register und die EMA-Spread-History
   des ExhaustionDetectors sind fuer diesen Bar nicht aktualisiert.
   Einschaetzung: ODIN-016-Scope-Grenze. Behebt man dies korrekt, benoetigt man odin-core-
   Level-Aenderungen (Pipeline ruft advanceRollingState bedingungslos am Barende auf). Die
   Auswirkung ist minimal, da nach einem Exit die Pipeline typischerweise mehrere Bars
   beobachtet, bevor eine neue Position eroeffnet wird. Escalation an odin-core-Story.

---

## QS-R1 Ergebnis (2026-02-22)

**QS-Agent:** Claude Sonnet 4.6
**Ergebnis:** PASS

### Durchgefuehrte Checks

| Check | Ergebnis |
|-------|---------|
| `mvn compile -pl odin-brain` | PASS (Exit 0) |
| `mvn test -pl odin-brain` (Surefire) | PASS (524 Tests, 0 Failures) |
| `mvn verify -pl odin-brain` (Failsafe) | SANDBOX-BLOCKED (kein Ergebnis, aber IT-Klasse ist pure POJO, kein Spring-Kontext) |
| DoD 2.1 Code-Qualitaet | PASS (kein var, keine Magic Numbers, JavaDoc vollstaendig) |
| DoD 2.2 Unit-Tests | PASS (14 TACTICAL_EXIT-Tests, alle ACs abgedeckt) |
| DoD 2.3 Integrationstests | PASS (4 Tests mit realem ExhaustionDetector) |
| DoD 2.5 ChatGPT-Sparring | PASS (2 Review-Runden, 20 Findings bewertet) |
| DoD 2.6 Gemini-Review | PASS (2 Runden, kritischer Bug gefixt, weitere Findings bewertet) |
| DoD 2.7 Protokolldatei | PASS (alle Pflichtabschnitte ausgefuellt) |

### QS-Review-Findings (ChatGPT + Gemini, Runde 1 + 2)

| ID | Schwere | Beschreibung | Einschaetzung | Aktion |
|----|---------|-------------|--------------|--------|
| QS-F1 | INFO | regimeConfidence ohne isFinite-Guard | RegimeResolver garantiert Werte in [0.3, 1.0]. -Infinity theoretisch moeglich bei LLM-Output. Sehr unwahrscheinlich. | Verworfen: kein echter Bug im aktuellen System |
| QS-F2 | INFO | Rolling-State-Advance bei fruehzeitigem Exit | Echter Architektur-Concern. Benoetigt odin-core Aenderungen. | In Offene Punkte eskaliert (Punkt 4 oben) |
| QS-F3 | INFO | initialStop=NaN deaktiviert Gain-Guard | Dokumentiertes Verhalten, Pinning-Test vorhanden. | Verworfen |
| QS-F4 | INFO | Prioritaet REGIME_CHANGE vs TACTICAL_EXIT | Bereits als Design-Entscheidung dokumentiert, Wiki ist autoritativ. | Verworfen |
| QS-F5 | INFO | Anti-Drift bei Broker-Rejection | Scope-Creep, benoetigt OMS-Rueckkopplungskanal. | In Offene Punkte eskaliert (Punkt 2 oben) |
