# QS-Report Round 1 — ODIN-015
**Datum:** 2026-02-21
**Pruefer:** QA-Agent (Claude Sonnet 4.6)
**Ergebnis:** FAIL

---

## Findings

| ID | Schwere | Kategorie | Beschreibung | Datei:Zeile |
|----|---------|-----------|--------------|-------------|
| F-001 | CRITICAL | Integration-Test-Failure | `warmupIncomplete_noTacticalExitEvenWithExtremeIndicators` schlaegt fehl. Test setzt `intradayHigh=200.0` und `currentPrice=151.0` (aus Bar.close). Dadurch faeuert `TRAILING_STOP` (197.8 > 151.0) — unabhaengig vom Warmup-Guard. `result.isPresent()` ist `true`, Test erwartet `false`. Root cause: Test-Snapshot hat unrealistisch hohes `intradayHigh` relativ zum aktuellen Preis, was einen Trailing-Stop-Fire ausloest, der nichts mit dem Exhaustion-Warmup-Guard zu tun hat. Der Warmup-Guard im `ExhaustionDetector` funktioniert korrekt — der Test deckt das falsche Szenario ab. | `ExhaustionDetectorIntegrationTest.java:272` |
| F-002 | MEDIUM | JavaDoc-Inkonsistenz | `TACTICAL_EXIT` JavaDoc sagt: "at least **2 of 3** exhaustion pillars". Die Implementierung erfordert jedoch **alle 3 Pillars** (AND-Semantik, `activePillarCount >= TOTAL_PILLARS` mit `TOTAL_PILLARS=3`). Die Implementierung folgt korrekt dem autoritativen Konzept (§7.4: `A AND B AND C`). Der JavaDoc ist falsch und widerspruecher dem Code und dem Konzept. | `ExitReason.java:22-23` |
| F-003 | LOW | Test-Count-Abweichung | Protocol dokumentiert 5 Integrationstests, tatsaechlich laufen 6 Tests (der `warmupIncomplete`-Test wurde hinzugefuegt aber nicht im Protocol gezaehlt). Kein fachlicher Bug, aber Dokumentationsluecke. | `protocol.md:85` / `ExhaustionDetectorIntegrationTest.java` |
| F-004 | INFO | Priority-Nummerierung | Konzept (§9) nummeriert TACTICAL_EXIT als Prio 5 (ohne KILL_SWITCH als explizite Prio). ExitRules und ExitReason nummerieren TACTICAL_EXIT als Prio 6 (KILL_SWITCH als Prio 1 eingefuegt). Die relative Reihenfolge ist korrekt. Nummerierungsabweichung durch KILL_SWITCH-Hinzufuegung — jedoch konsistent dokumentiert in ExitRules-JavaDoc und ExitReason-Klassen-JavaDoc. Kein Bug, aber leichte Verwirrung beim Konzeptvergleich. | `ExitReason.java:8-9` / `ExitRules.java:22-24` |

---

## Testlauf-Ergebnisse

### Unit-Tests (Surefire) — `mvn test -pl odin-brain`
```
Tests run: 341, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
Total time: 4.371 s
```
341 Unit-Tests GRUEN. Alle 22 neuen ExhaustionDetectorTest-Tests bestanden.

### Integration-Tests (Failsafe) — `mvn verify -pl odin-brain -am`
```
Tests run: 6, Failures: 1, Errors: 0, Skipped: 0

FAILURE: warmupIncomplete_noTacticalExitEvenWithExtremeIndicators
AssertionFailedError: Warmup-incomplete must prevent TACTICAL_EXIT even with extreme indicators
==> expected: <false> but was: <true>

BUILD FAILURE
```
1 von 6 Integrationstests FEHLGESCHLAGEN.

---

## Detailanalyse: F-001 (Root Cause)

Test-Aufbau:
```java
Bar wideBar = buildBar(148.0, 152.0, 147.0, 151.0, 50000);
// intradayHigh = 200.0 (viel hoeher als Preis!)
MarketSnapshot snapshot = buildSnapshotWithBars(List.of(wideBar), 200.0);
```

Ergebnis in `ExitRules.checkExit()`:
- KILL_SWITCH: nein (kein CRASH_SIGNAL-Flag)
- FORCED_CLOSE: nein (state=POSITIONED)
- STOP_LOSS: nein (currentPrice=151 > stopLevel=150-1.5*1.0=148.5)
- TRAILING_STOP: `trailingAtr = currentAtr = 1.0` (weil `entryAtr=NaN` in 7-arg overload)
  - `candidateTrailingStop = 200 - 2.2*1.0 = 197.8`
  - `currentPrice=151 <= 197.8` => TRAILING_STOP FIRES

Der ExhaustionDetector wird gar nicht erst aufgerufen — TRAILING_STOP hat hoehere Prioritaet. Die Warmup-Pruefung des ExhaustionDetectors ist korrekt implementiert, wird aber in diesem Test nicht erreicht. **Der Test deckt nicht was er zu testen vorgibt.**

---

## Detailanalyse: F-002 (JavaDoc-Bug)

Konzept §7.4 (autoritativ):
```
exhaustion_kpi_confirmed = (Saeule_A_has_true) AND (Saeule_B_has_true) AND (Saeule_C_has_true)
```

Implementierung (korrekt):
```java
private static final int TOTAL_PILLARS = 3;
boolean confirmed = activePillarCount >= TOTAL_PILLARS;  // == 3
```

JavaDoc TACTICAL_EXIT (falsch):
```java
// "at least 2 of 3 exhaustion pillars"  <-- FALSCH
```

Die Implementierung ist korrekt. Nur der JavaDoc ist falsch (haelt den alten Story-Wortlaut "2 von 3" statt des Konzepts "alle 3").

---

## Erfuellte DoD-Kriterien

### 2.1 Code-Qualitaet — PASS
- [x] Kein `var` — explizite Typen ueberall in ExhaustionDetector.java
- [x] Keine Magic Numbers — alle Schwellen sind `private static final` Konstanten (TOTAL_PILLARS=3, LOOKBACK_BARS=2)
- [x] JavaDoc auf allen public Klassen und Methoden (ExhaustionDetector, detectExhaustion, ExitRules-Klasse, ExitSignal-Record)
- [x] Verb-first Methodennamen: `detectExhaustion`, `evaluatePillarA/B/C`, `advanceRollingState`, `resolveLatestBar`
- [x] Records fuer DTOs: ExitSignal als Record
- [x] ENUM korrekt eingesetzt: ExitReason als Enum mit TACTICAL_EXIT
- [x] Keine TODO/FIXME-Kommentare
- [x] Code-Sprache Englisch
- [x] Namespace `odin.brain.rules.exhaustion.*` in odin-brain.properties korrekt

### 2.2 Tests Klassenebene — PASS (22 Unit-Tests)
- [x] Extension Pillar einzeln getestet
- [x] Climax Pillar einzeln getestet
- [x] Rejection Pillar einzeln getestet
- [x] 2-von-3-Logik (tatsaechlich: 3-von-3 Consensus) getestet
- [x] Integration in ExitRules getestet (Exhaustion triggert vor Target-Check)
- [x] Doji (high==low, range=0): kein Division-durch-Null, Pillar-B nicht getriggert
- [x] Warmup-incomplete → false (Unit-Test-Ebene besteht)
- [x] ATR=NaN → false
- [x] ATR=0 → false
- [x] Konvention `*Test` (Surefire) eingehalten

### 2.3 Tests Komponentenebene — FAIL (6 Integrationstests, 1 schlaegt fehl)
- [x] ExhaustionDetector mit ExitRules zusammengeschaltet
- [x] E2E-Flow Exhaustion → TACTICAL_EXIT
- [x] Reihenfolge TRAILING_STOP > Exhaustion > TARGET bestaetigt
- [x] RSI-State-Propagation ueber mehrere ExitRules.checkExit-Calls
- [FAIL] Warmup-Guard-Integrationstest schlaegt fehl (F-001)

### 2.4 Tests Datenbank — N/A (kein DB-Zugriff in dieser Story)

### 2.5 Test-Sparring ChatGPT — PASS
- [x] ChatGPT-Review dokumentiert in protocol.md
- [x] Integration-Test-Schwaeche behoben (assertFalse auf isPresent statt ifPresent)
- [x] "Always-exhausted"-Stub-Non-Determinismus behoben
- [x] Findings bewertet und in protocol.md dokumentiert

### 2.6 Review Gemini (3 Dimensionen) — PASS
- [x] Dimension 1 Code-Bugs: CRITICAL RSI-Shift-Register-Bug gefunden und gefixt
- [x] Dimension 2 Konzepttreue: PASS dokumentiert
- [x] Dimension 3 Praxis: Long-only-Bias, Frozen-ATR, Session-Phase-Guard dokumentiert
- [x] RSI-Shift-Register-Bug-Fix dokumentiert (protocol.md §Key Design Decisions #3)

### 2.7 Protokolldatei — PASS
- [x] protocol.md vorhanden und ausgefuellt
- [x] Design-Entscheidungen dokumentiert (3-of-3 vs 2-of-3, RSI-Shift-Register, Priority-Order)
- [x] ChatGPT-Sparring-Abschnitt ausgefuellt
- [x] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss — PASS
- [x] Commit vorhanden: `6b2596c feat(odin-015): implement 3-pillar exhaustion detection with TACTICAL_EXIT`
- [x] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`
- [ ] Integration-Test besteht nicht → DoD 2.3 nicht erfuellt

---

## Akzeptanzkriterien-Auswertung

| Kriterium | Status | Bemerkung |
|-----------|--------|-----------|
| Extension Pillar (A1: RSI, A2: Bollinger, A3: VWAP) | PASS | Alle 3 Teilkriterien implementiert |
| Climax Pillar (B1: Volume, B2: Range) | PASS | Beide Teilkriterien implementiert |
| Rejection Pillar (C1: BB Re-Entry, C2: VWAP Snapback, C3: Fast RSI Drop, C4: Trend Stall) | PASS | Alle 4 Teilkriterien implementiert — reicher als Story-Spec |
| Exhaustion = mindestens 2 von 3 (Story) / alle 3 (Konzept) | PASS* | Impl. folgt autoritativem Konzept (alle 3) korrekt; Story-Wortlaut war veraltet |
| Integration in ExitRules VOR Target-Check | PASS | Prio 6 TACTICAL_EXIT vor Prio 7 TARGET |
| ExitReason TACTICAL_EXIT | PASS | Vorhanden, aber JavaDoc falsch (F-002) |
| Alle Schwellen konfigurierbar in BrainProperties | PASS | 7 Properties in ExhaustionProperties-Record |

---

## Ergebnis-Zusammenfassung

**FAIL** wegen:

1. **F-001 (CRITICAL):** Integrations-Test `warmupIncomplete_noTacticalExitEvenWithExtremeIndicators` schlaegt fehl. Der ExhaustionDetector-Warmup-Guard ist korrekt implementiert, aber der Test prueft ihn falsch (Trailing-Stop faeuert statt Exhaustion-Exit, weil intradayHigh=200 waehrend currentPrice=151). Test muss korrigiert werden.

2. **F-002 (MEDIUM):** TACTICAL_EXIT JavaDoc behauptet "at least 2 of 3" aber korrekte Semantik ist "alle 3". Verwirrung fuer Leser.

Die Kernimplementierung des ExhaustionDetectors ist fachlich und architektonisch korrekt. Die Unit-Tests (341) laufen durch. Nur die Integrations-Test-Suite hat einen Defekt.
