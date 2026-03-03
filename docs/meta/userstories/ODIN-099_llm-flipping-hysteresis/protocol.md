# ODIN-099 — LLM-Flipping-Hysterese: Protocol

> **Story-ID:** ODIN-099
> **Status:** Done
> **Implementiert:** 2026-03-03
> **Modul:** odin-brain

---

## 1. Working State

### Implementierte Dateien

| Datei | Typ | Beschreibung |
|-------|-----|--------------|
| `odin-brain/src/main/java/de/its/odin/brain/llm/HysteresisFilter.java` | NEW | Generischer Hysterese-Filter (Option A) |
| `odin-brain/src/main/java/de/its/odin/brain/llm/LlmAnalystOrchestrator.java` | MODIFIED | 3 Filter-Felder, resetForNewSession(), buildTacticalOutput() |
| `odin-brain/src/test/java/de/its/odin/brain/llm/HysteresisFilterTest.java` | NEW | 25 Unit-Tests |
| `odin-brain/src/test/java/de/its/odin/brain/llm/HysteresisFilterIntegrationTest.java` | NEW | 9 Integrationstests |

### Test-Ergebnisse

```
HysteresisFilterTest:            25 tests, 0 failures (Surefire)
HysteresisFilterIntegrationTest:  9 tests, 0 failures (Failsafe)
odin-brain total unit tests:   1685 tests, 0 failures
```

Bekannte Pre-existing Failures (vor ODIN-099, nicht durch diese Story verursacht):
- `ExhaustionDetectorIntegrationTest`: 3 failures
- `OpeningConsolidationFsmIntegrationTest`: 1 failure
- `ReEntryCorrectionFsmIntegrationTest`: 1 failure

Diese Failures betreffen ExhaustionDetector und Pattern-FSM-Klassen, die keinerlei Bezug zu
`LlmAnalystOrchestrator`, `HysteresisFilter` oder den hier geaenderten Klassen haben.

---

## 2. Design-Entscheidungen

### D1: Option A — Generisches HysteresisFilter<T>

Alle drei Felder (trailMode, entryTiming, action) benoetigen exakt dieselbe 2-konsekutiv-Logik.
Option A (generischer Filter) wurde gewaehlt, da:
- Kein Over-Engineering: exakt 3 Nutzer, identische Logik
- Single Point of Change fuer REQUIRED_CONSECUTIVE
- Testbarkeit: reine Zustandsmaschine, keine Spring-Abhaengigkeiten

### D2: trail_mode als Neuzusatz (nicht nur Erweiterung)

Die Analyse ergab: §13 des Konzepts beschreibt Hysterese fuer trail_mode, aber im tatsaechlichen
Code existierte diese Logik nicht vor ODIN-099. ODIN-099 implementiert trail_mode-Hysterese erstmalig
und konsistent ueber denselben generischen Filter wie entryTiming und action.

### D3: Session-Isolation durch frische Instanz

`LlmAnalystOrchestrator` wird pro Trading-Pipeline-Lauf neu instanziiert (kein Spring-Singleton).
Dadurch ist Session-Isolation automatisch gewaehrleistet. Zusaetzlich wurde `resetForNewSession()`
als explizite Methode hinzugefuegt fuer Multi-Day-Simulation-Szenarien.

### D4: action=EXIT nicht ausgenommen

ChatGPT-Sparring empfahl, EXIT vom Hysterese-Filter auszunehmen (sofortiger Exit aus Risk-Management-
Gruenden). Nach Analyse der ExitRules.java wurde festgestellt: TACTICAL_EXIT erfordert bereits
multiple Quant-Guardrails (min. 0.5R Profit, MIN_HOLD_BARS, Exhaustion-Pillars, Max-Tactical-Exits-
Counter). Der LLM-EXIT ist also kein "Panic Exit", sondern ein kontrollierter Austritt mit mehrfacher
Absicherung. Die 2-Bar-Verzoegerung ist daher akzeptabel und entspricht der Story-Spezifikation.
Als offenes Item fuer zukuenftige Optimierung notiert (asymmetrische Hysterese).

### D5: requireNonNull mit Lambda-Supplier

Auf Empfehlung von ChatGPT-Sparring wurde die String-Konkatenation im `requireNonNull`-Aufruf von
eagerer Auswertung auf lazy Lambda-Supplier umgestellt:
`Objects.requireNonNull(newValue, () -> "newValue must not be null for field: " + fieldName);`

### D6: Zwei zusaetzliche Tests auf ChatGPT-Empfehlung

Auf Basis des ChatGPT-Sparrings wurden zwei neue Unit-Tests hinzugefuegt:
- `interruptedCandidate_candidateInterruptedByCurrentValue_mustRestartCount`
- `candidateReplacedRightBeforeConfirmation_newCandidateDoesNotInheritProgress`
Diese testen explizit das korrekte Counter-Reset-Verhalten nach Kandidaten-Unterbrechung.

---

## 3. Offene Punkte

### O1: Asymmetrische Hysterese fuer action=EXIT (Future Optimization)

ChatGPT-Sparring und Gemini-Praxis-Review empfehlen uebereinstimmend: EXIT sollte langfristig
den Hysterese-Filter bypassen (sofortige Wirkung) waehrend ENTER/HOLD/NO_TRADE weiterhin
2-konsekutiv erfordern.

Begruendung: Eine 2-Bar-Verzoegerung auf EXIT kann in einem 1-Minuten-Bar-System 2 Minuten
bedeuten — in einem V-Reversal oder News-Event kann dies signifikant sein.

Gegenaargument: TACTICAL_EXIT hat Quant-Guardrails (min 0.5R, exhaustion pillars), die den
LLM-EXIT als letzten Filter einsetzen. Ein Hard-Stop in der Execution-Schicht waere der robustere
Ansatz.

**Empfehlung:** Separat als ODIN-1XX Micro-Story anlegen: "Asymmetrische Hysterese: EXIT bypass".
Scope von ODIN-099: unveraendert nach Story-Spezifikation.

### O2: HysteresisFilter als Bounded Type T extends Enum<T> (Low Priority)

ChatGPT-Feedback: Fuer eine Domain-Klasse waere `T extends Enum<T>` sicherer als unbounded `T`.
Entschieden: Unbounded `T` beibehalten, da der Filter theoretisch auch mit anderen Typen
(Records, Strings) verwendbar sein koennte. Verwendung von `equals()` statt `==` ist korrekt.

---

## 4. ChatGPT-Sparring (DoD 5.5)

**Session:** 2026-03-03, owner=ODIN-099-worker

**Themen:** Hysterese-Logik, Edge Cases, Thread-Safety, action=EXIT-Risiko, Test-Luecken

### Key Findings

**Finding 1 (Berechtigte Empfehlung — umgesetzt):** requireNonNull eager String-Concat
→ Lambda-Supplier verwenden: `Objects.requireNonNull(newValue, () -> "...")`. Umgesetzt.

**Finding 2 (Berechtigte Empfehlung — umgesetzt):** Test "Interrupted Candidate" fehlt.
Sequenz: current=A, apply(B)(1), apply(A)(clear), apply(B)(1 — nicht 2!), apply(B)(confirm).
Ohne expliziten Test koennte ein Refactoring den Counter nicht korrekt zuruecksetzen.
Zwei neue Tests hinzugefuegt: `interruptedCandidate_...` und `candidateReplacedRightBefore...`.

**Finding 3 (Bewertet — nicht umgesetzt in ODIN-099):** EXIT vom Filter ausnehmen.
Begruendung fuer Nicht-Umsetzung: TACTICAL_EXIT hat Quant-Guardrails; Story-Scope unveraendert.
Als O1 dokumentiert fuer kuenftige Optimierung.

**Finding 4 (Kein Handlungsbedarf):** Thread-Safety: volatile oder synchronized empfohlen.
LlmAnalystOrchestrator ist ein per-pipeline POJO (nicht Spring-Singleton). Thread-Confinement
ist garantiert durch Pipeline-Architektur. Volatile wuerde architekturellen Missbrauch verstecken.

**Finding 5 (Kein Handlungsbedarf):** T extends Enum<T> vs unbounded T.
Entschieden: unbounded beibehalten. Siehe O2.

---

## 5. Gemini-Review (DoD 5.6)

### Dimension 1: Code-Review

**Session:** 2026-03-03, owner=ODIN-099-gemini-1

**Bewertung:** Zustandsautomat korrekt.

Key Findings:
- consecutiveCount kann nicht > 1 im Steady-State steigen (korrekt: Reset bei Bestaetigung)
- State Leak: Nach Bestaetigung ist candidate=null, count=0. Kein Leak moeglich.
- Symmetrie: Rueckwechsel zum alten Wert erfordert ebenfalls 2 konsekutive — korrekt (Hysterese)
- requireNonNull Lambda-Supplier: Java-8+-Standard, korrekt in Java 21
- T vs T extends Enum: Unbounded korrekt wenn equals() verwendet wird

Keine Code-Fehler gefunden. Alle State-Machine-Transitionen verifiziert.

### Dimension 2: Konzepttreue-Review

**Session:** 2026-03-03, owner=ODIN-099-gemini-2

**Alle 5 Pruefpunkte: YES**

| Pruefpunkt | Ergebnis |
|------------|----------|
| "2x konsekutiv" bedeutet Aenderung bei 2. Vorkommen | YES — korrekte Interpretation |
| 3x-Kriterium: spaetestens beim 2. Mal | YES — automatisch erfuellt |
| Session-Isolation durch frische Instanz | YES — robuster Dual-Ansatz |
| Option A (generisch) konzeptkonform | YES — DRY, single point of change |
| trail_mode als Neuzusatz korrekt | YES — Konzept §13 war nicht implementiert |

### Dimension 3: Praxis-Review

**Session:** 2026-03-03, owner=ODIN-099-gemini-3

**Bewertung:** 4 Szenarien analysiert.

| Szenario | Befund | Handlungsbedarf |
|----------|--------|----------------|
| ORB Opening: entryTiming-Flip absorbiert | Sub-optimal fuer Breakouts, aber Opening-Phase-spezifische Logik ist Out-of-Scope ODIN-099 | Nein (ODIN-098 handhabt Sampling-Policy) |
| EXIT-Verzoegerung 2 Bars | Akzeptabel fuer profit-taking da Quant-Guardrails vorhanden; problematisch fuer loss-cutting ohne Hard-Stop | Als O1 notiert |
| Stickiness (WAIT_PULLBACK persistiert) | Feature: reduziert Over-Trading in Seitwärtsmaerkten | Kein Handlungsbedarf |
| Kein resetForNewSession: "Ghost Signals" | KRITISCH: stale action=EXIT vom Vortag wuerde sofort triggered | Kein Handlungsbedarf — per-instance-creation garantiert Reset, explicit reset() zusaetzlich vorhanden |

**Fazit:** Kein Handlungsbedarf auf Basis der Gemini-Reviews. Alle Findings entweder
already-handled oder als O1 fuer kuenftige Optimierung dokumentiert.
