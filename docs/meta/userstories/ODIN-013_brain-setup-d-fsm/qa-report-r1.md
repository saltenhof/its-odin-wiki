# QA Report R1 — ODIN-013: ReEntryCorrectionFsm

**QA-Agent:** Claude Sonnet 4.6
**Datum:** 2026-02-22
**Ergebnis:** FAIL

---

## Zusammenfassung

Die Implementierung ist in Struktur, Codestil, Tests und Architektur-Konformität sehr solide. Kompilierung und alle 476 Unit-Tests sowie 3 Integration-Tests laufen fehlerfrei durch. Dennoch wurden im ChatGPT- und Gemini-Review zwei kritische/schwerwiegende Fehler identifiziert, die die Korrektheit der Fibonacci-Berechnungen in realen Marktbedingungen gefährden.

---

## Checkliste-Ergebnisse

| Schritt | Status | Details |
|---------|--------|---------|
| Kompilierung (`mvn compile -pl odin-brain`) | PASS | 0 Fehler |
| Unit-Tests (Surefire, 14 Tests) | PASS | 0 Failures |
| Integration-Tests (Failsafe, 3 Tests) | PASS | 0 Failures (direkt via Surefire aufgerufen) |
| Gesamtes odin-brain Test-Suite (476 Tests) | PASS | 0 Failures |
| DoD-Check (Akzeptanzkriterien) | PARTIAL | Siehe Finding F1 (SIGNAL_ACTIVE fehlt — akzeptables Design-Abweichen) |
| Code-Review (CLAUDE.md) | PASS | Keine Magic Numbers, keine var, JavaDoc vollständig, Enums für States, Records für DTOs |
| ChatGPT-Review (2 Runden) | FAIL | CRITICAL: stale swingLowBeforeRun; MAJOR: fehlender currentPrice-Guard |
| Gemini-Review (3 Dimensionen) | FAIL | CRITICAL: recentLows nicht bei Lower-Low gecleart; MAJOR: Swing-High-Ankerpunkt; MAJOR: fehlender Test für choppy-prelude |

---

## Findings

### F1 — DESIGN-ABWEICHUNG (Akzeptabel): Fehlender SIGNAL_ACTIVE-State

**Beschreibung:** Die Story fordert 4 FSM-States (IDLE, TREND_ESTABLISHED, PULLBACK_ACTIVE, SIGNAL_ACTIVE). Die Implementierung hat nur 3 — SIGNAL_ACTIVE fehlt. Das Signal wird emittiert und direkt zu IDLE resettet.

**Bewertung:** Beide Reviewer (ChatGPT und Gemini) bestätigen: Für event-driven Trading-Engines ist dies ein besseres Design. Kein "stuck in signal state", keine doppelten Orders. SIGNAL_ACTIVE als Transient-State ohne echte Logik bringt keinen Mehrwert.

**Entscheidung:** Akzeptiert. Die JavaDoc erklärt das Verhalten korrekt als "transient". Keine Änderung notwendig.

---

### F2 — CRITICAL: `recentLows.getFirst()` liefert stale Trend-Ursprung

**Datei:** `ReEntryCorrectionFsm.java`, Zeile 208
**Code:**
```java
swingLowBeforeRun = recentLows.getFirst();
```

**Problem:** Die `recentLows`-LinkedList ist ein rolling 20-Bar-Window. Wenn `confirmedHigherLows` wegen eines Lower-Lows auf 0 zurückgesetzt wird (Zeile 191), wird die `recentLows`-Liste NICHT gecleart. Szenario:
- Bars 1–16: choppy (mehrfache Lower-Lows, Counter resettet wiederholt)
- Bars 17–20: 3 konsekutive Higher-Lows → TREND_ESTABLISHED
- `recentLows.getFirst()` gibt das Low von Bar 1 zurück — NICHT von Bar 17 (dem echten Trend-Ursprung)

Dies verzerrt `upMoveSize = swingHighBeforePullback - swingLowBeforeRun` und damit alle Fibonacci-Levels. Im schlimmsten Fall:
- Fibonacci-Zone liegt an der falschen Stelle → falsche PULLBACK_ACTIVE-Übergänge
- Breakdown-Threshold (61.8%) liegt falsch → fehlendes Invalidation-Signal
- Confidence-Berechnung basiert auf falschen Werten

**Bestätigung:** Sowohl ChatGPT als auch Gemini markieren dies unabhängig voneinander als CRITICAL.

**Fix:** Wenn `confirmedHigherLows = 0` gesetzt wird, `recentLows` leeren und nur das aktuelle Low einfügen:
```java
if (latestLow > previousLow) {
    confirmedHigherLows++;
} else {
    confirmedHigherLows = 0;
    // Clear history — trend origin tracking resets with the higher-low sequence
    recentLows.clear();
    recentLows.addLast(latestLow);  // keep current low as potential new series start
}
```

**Pflicht:** Test hinzufügen, der diesen Szenario abdeckt (choppy prelude + 3 HL → korrekte Fibonacci-Zone).

---

### F3 — MAJOR: Fehlender Guard für ungültigen `currentPrice`

**Datei:** `ReEntryCorrectionFsm.java`, Zeile 140
**Code:**
```java
double currentPrice = snapshot.currentPrice();
```

**Problem:** In `onBar()` wird `atr` auf NaN/<=0 geprüft (Zeile 143–145), aber `currentPrice` wird ohne Validierung verwendet. Wenn `snapshot.currentPrice()` NaN oder 0 zurückgibt (Feed-Initialisierung, Datenfehler), können die ATR-Timeout-Checks und Fibonacci-Berechnungen fehlerhafte Ergebnisse produzieren oder spurious State-Transitions auslösen.

**Fix:** Guard vor dem Switch-Statement hinzufügen:
```java
if (!Double.isFinite(currentPrice) || currentPrice <= 0) {
    return Optional.empty();
}
```

---

### F4 — MAJOR: Fehlender Regressions-Test für Choppy-Prelude-Szenario

**Datei:** `ReEntryCorrectionFsmTest.java`

**Problem:** Alle Unit-Tests verwenden monoton steigende Lows (0, 1, 2, ... höher). Der Fehler aus F2 würde von diesen Tests NICHT gefunden, weil es keine Lower-Low-Resets gibt. Ein Test, der explizit mehrere Lower-Low-Resets vor den 3 konsekutiven Higher-Lows simuliert und dann die Korrektheit der Fibonacci-Zone prüft, fehlt.

**Fix:** Test hinzufügen:
- Bars 0–3: choppy (Lower-Low tritt auf, Counter resettet)
- Bars 4–7: 3 konsekutive Higher-Lows → TREND_ESTABLISHED
- Prüfen, dass `swingLowBeforeRun` dem Low von Bar 4 entspricht (nicht Bar 0)
- Pullback in korrekte Fibonacci-Zone → PULLBACK_ACTIVE

---

### F5 — MINOR (Bestätigt akzeptabel): Swing-High-Ankerpunkt kann bei laggendem ADX suboptimal sein

**Datei:** `ReEntryCorrectionFsm.java`, Zeile 209
**Code:**
```java
swingHighBeforePullback = currentBar.high();
```

**Problem:** Wenn das absolute Swing-High einen Bar vor dem ADX-Threshold-Übertritt liegt, wird der echte Peak verpasst. Da die TREND_ESTABLISHED-State den High kontinuierlich nach oben aktualisiert (`if (currentBar.high() > swingHighBeforePullback)`), ist der Effekt begrenzt — der echte Peak wird nachgeholt, sobald der Markt wieder darüber steigt.

**Entscheidung:** MINOR — das ist kein kritischer Bug, da der High kontinuierlich aktualisiert wird. Der initiale "niedrigere" High führt nur kurzzeitig zu einer etwas engeren Fibonacci-Zone, die sich selbst korrigiert. Keine Änderung notwendig, aber dokumentieren.

---

## Fazit

**Zwei CRITICAL/MAJOR-Fixes sind erforderlich:**
1. **F2 (CRITICAL):** `recentLows` bei Lower-Low-Reset leeren + Regressions-Test
2. **F3 (MAJOR):** `currentPrice` auf `Double.isFinite() && > 0` prüfen

**DoD nicht erfüllt:**
- [ ] Bugfix → reproduzierender Test (F2 erfordert neuen Test) — PFLICHT laut DoD 2.2
- [ ] Alle Akzeptanzkriterien korrekt implementiert (F2 verletzt die Korrektheit der Fibonacci-Berechnung)

**Empfehlung:** Story in R2 mit den konkreten Findings als Input zurückgeben. Fix-Umfang ist klein (< 30 Zeilen Code + 1 neuer Test).
