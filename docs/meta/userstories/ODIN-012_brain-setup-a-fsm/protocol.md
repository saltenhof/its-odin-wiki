# ODIN-012 Implementation Protocol
## Setup A: Opening Consolidation → Trend Pattern FSM

**Status:** QS PASS — COMMITTED
**Datum:** 2026-02-22
**Bearbeiter:** Implementierungs-Agent → QS-Agent (Claude Sonnet 4.6)

---

## 1. Arbeitszustand

### Implementierte Dateien

| Datei | Änderung |
|-------|----------|
| `odin-brain/src/main/java/de/its/odin/brain/rules/pattern/OpeningConsolidationFsm.java` | NEU — vollständige FSM-Implementierung |
| `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` | GEÄNDERT — PatternProperties um `consolidationRangeAtrFactor` und `volumeConfirmationMultiplier` erweitert |
| `odin-brain/src/main/resources/odin-brain.properties` | GEÄNDERT — zwei neue Properties-Defaults hinzugefügt |
| `odin-brain/src/test/java/de/its/odin/brain/TestBrainProperties.java` | GEÄNDERT — PatternProperties-Konstruktor auf 10 Felder aktualisiert |
| `odin-brain/src/main/java/de/its/odin/brain/rules/RulesEngine.java` | GEÄNDERT — OpeningConsolidationFsm registriert in `initializePatternFsms()` |
| `odin-brain/src/test/java/de/its/odin/brain/rules/pattern/OpeningConsolidationFsmTest.java` | NEU — 27 Unit Tests |
| `odin-brain/src/test/java/de/its/odin/brain/rules/pattern/OpeningConsolidationFsmIntegrationTest.java` | NEU — 4 Integrationstests |

### Test-Ergebnis

```
Tests run: 68, Failures: 0, Errors: 0, Skipped: 0
- OpeningConsolidationFsmTest: 27 Tests
- OpeningConsolidationFsmIntegrationTest: 4 Tests
- CoilBreakoutFsmTest: 9 Tests (Regression: grün)
- FlushReclaimRunFsmTest: 7 Tests (Regression: grün)
- RulesEngineTest: 21 Tests (Regression: grün)
BUILD SUCCESS
```

**Hinweis:** `IntentSignerVerifierIntegrationTest` hat einen pre-existing Kompilierfehler (fehlende Klasse `IntentVerifier` aus `odin-execution`). Dieser Fehler existierte vor diesem Story-Start und ist nicht durch ODIN-012 verursacht. Tests wurden mit temporärem Rename dieser Datei ausgeführt.

---

## 2. Design-Entscheidungen

### 2.1 Opening-Window Counter (barsSinceRthOpen) — Trennung von Pattern-State

**Problem:** Eine naiver Counter, der bei `resetFields()` zurückgesetzt wird, erlaubt es dem Pattern, nach einer Invalidierung (z.B. Preis unter consolidationLow) den Opening-Window-Timer neu zu starten und damit nach >60 Bars noch ein Signal zu erzeugen.

**Lösung:** Zwei separate Mechanismen:
- `barsSinceRthOpen`: Globaler Counter für Bars seit RTH_OPENING-Start. Wird nur bei `reset()` (expliziter Reset, z.B. neuer Tag) oder bei Phase-Transition aus RTH_OPENING heraus zurückgesetzt.
- `openingWindowExpired`: Latch. Sobald `barsSinceRthOpen > 60`, wird `openingWindowExpired = true` gesetzt und die FSM akzeptiert keine weiteren Bars im aktuellen Opening.
- `resetPatternState()`: Setzt nur state, consolidationHigh, consolidationLow, breakoutLevel zurück — berührt den Timer NICHT.

### 2.2 SessionPhase-Tracking (lastSessionPhase)

Um Phase-Transitionen (Enter/Leave RTH_OPENING) sauber zu erkennen, wird `lastSessionPhase` mitgeführt. Bei Verlassen von RTH_OPENING: vollständiger Reset. Bei Betreten von RTH_OPENING aus einer anderen Phase: vollständiger Reset (Opening-Window startet neu).

### 2.3 breakoutLevel als persistentes Feld

Beim Übergang CONSOLIDATING → BREAKOUT_PENDING wird `breakoutLevel = consolidationHigh + breakoutOffset` gespeichert. In BREAKOUT_PENDING wird nur bestätigt, wenn `close >= breakoutLevel AND volumeRatio >= threshold`. Damit ist sichergestellt, dass der Breakout "intakt" ist (Preis über dem Trigger-Level) und nicht nur irgendeine Bar mit hohem Volumen innerhalb der Range die Confirmation auslöst.

### 2.4 Revalidierung in CONSOLIDATING

Beim Expandieren der Bounds in CONSOLIDATING wird nach jedem Bar geprüft, ob `range >= consolidationRangeAtrFactor * ATR`. Falls ja, resetPatternState(). Damit wird verhindert, dass eine Konsolidierungsphase, die sich ausweitet, fälschlicherweise weiter als "tight" angesehen wird.

**Evaluationsreihenfolge:** Invalidation → Breakout-Check (gegen altes High) → Bounds-Expansion → Revalidierung. Damit wird der Breakout gegen das "echte" Konsolidierungs-High bewertet, bevor das aktuelle Bar's High die Bounds manipuliert.

### 2.5 Rückfall BREAKOUT_PENDING → CONSOLIDATING

Wenn Preis unter consolidationHigh fällt (Breakout nicht mehr intakt, aber noch über consolidationLow):
- Aktuelles Bar's High/Low wird in die Bounds eingearbeitet
- Range wird revalidiert: wenn zu breit → IDLE, sonst → CONSOLIDATING
- Damit bleiben die Bounds konsistent mit dem tatsächlichen Preisverlauf

### 2.6 Konsistente Preisquelle: Bar.close()

Alle State-Transitions verwenden `bars.getLast().close()`. Dies verhindert Flatter durch intrabar-Schwankungen und ist konsistent mit dem bar-basierten Aufrufmodell (1-Minuten-Bars in `decisionBars`).

### 2.7 Konfigurierbare vs. hardcodierte Parameter

Ins `config` aufgenommen: `consolidationRangeAtrFactor`, `volumeConfirmationMultiplier` (beide haben starken Tuning-Bedarf je nach Instrument/Volatilität).

Als Konstanten beibehalten: `CONSOLIDATION_LOOKBACK_BARS=5`, `BREAKOUT_OFFSET_ATR_FACTOR=0.1`, `OPENING_WINDOW_BARS=60`. Diese sind konzeptionell festgelegt (60 Minuten RTH Opening, 5-Bar-Lookback ist Standard für Opening-Setups) und werden vorerst nicht konfigurierbar gemacht.

---

## 3. Offene Punkte / Nicht umgesetzte Findings

| Finding | Quelle | Status | Begründung |
|---------|--------|--------|------------|
| Alle Strategy-Parameter in config | ChatGPT F8 | Nicht umgesetzt | CONSOLIDATION_LOOKBACK_BARS, BREAKOUT_OFFSET_ATR_FACTOR, OPENING_WINDOW_BARS sind konzeptionell fixe Werte. Separate Story wenn Bedarf. |
| PatternSignal Direction/Side-Feld | Gemini D3 | Nicht umgesetzt | PatternSignal ist shared record, Änderung erfordert eigene Story. ODIN ist Long-Only. |
| PatternSignal Timestamp | Gemini D3 | Nicht umgesetzt | Execution Engine hat MarketSnapshot mit Timestamp. Nicht im FSM-Scope. |
| SIGNAL_ACTIVE für mehrere Bars beibehalten | ChatGPT F6 | Nicht umgesetzt | SIGNAL_ACTIVE ist bewusst transient. Extern nicht sinnvoll beobachtbar, FSM setzt sofort zurück. |
| Intrabar-Breakout statt close-basiert | Gemini D2 | Nicht umgesetzt | Design-Entscheidung: close-basiert reduziert Wick-Traps und Spoofing. Für 1-min-Bars validiert. |
| Unidirektional (Long-Only) | Gemini D2 | Nicht umgesetzt | By design. ODIN ist Long-Only per Architektur. |

---

## 4. ChatGPT-Sparring

### Runde 1 — Code Review (19 Findings)

ChatGPT hat einen umfassenden Review durchgeführt und 19 Findings identifiziert. Kritischste Punkte:

**Finding 1 (CRITICAL): openingBarCount durch resetFields() umgehbar**
→ Implementiert: `barsSinceRthOpen` + `openingWindowExpired` Latch, vollständig entkoppelt von Pattern-Resets.

**Finding 3 (IMPORTANT): BREAKOUT_PENDING ohne "Breakout intakt" Guard**
→ Implementiert: `breakoutLevel` als persistentes Feld; Rückfall auf CONSOLIDATING wenn Preis < consolidationHigh.

**Finding 4 (IMPORTANT): Keine Revalidierung der Konsolidierungsrange**
→ Implementiert: Nach Bounds-Expansion in CONSOLIDATING wird `range >= threshold` geprüft; bei Überschreitung resetPatternState().

**Finding 5 (IMPORTANT): Inkonsistente Preisquelle**
→ Implementiert: Alle Transitions verwenden `bars.getLast().close()` (nicht snapshot.currentPrice()).

**Finding 6 (HINT): buildSignal currentPrice-Parameter ungenutzt**
→ Implementiert: currentPrice-Parameter aus buildSignal entfernt.

**Finding 9 (IMPORTANT): breakoutLevel nicht persistent**
→ Implementiert: `breakoutLevel` als Instanzfeld, gesetzt bei CONSOLIDATING→BREAKOUT_PENDING.

**Finding 2 (IMPORTANT): SessionPhase-Exit nicht zuverlässig wenn state==IDLE**
→ Implementiert: `lastSessionPhase` Tracking; bei Leave/Enter RTH_OPENING immer vollständiger Reset.

### Runde 2 — Konkrete Fix-Strategie

ChatGPT hat konkrete Code-Snippets für alle drei kritischen Fixes geliefert und "expandierende Bounds + Revalidierung" gegenüber Sliding-Window empfohlen (für Opening-Setups praktikabler). Empfehlung wurde übernommen.

---

## 5. Gemini-Review

### Dimension 1 — Code-Bugs

**Bug 1 (Design, kein Bug): Volume auf Breakout-Bar vs. Next-Bar**
→ Nicht geändert. Design-Entscheidung für 1-Minuten-Bars: next-bar Volume-Confirmation filtert Exhaustion-Spikes besser.

**Bug 2 (REAL BUG): State-Corruption beim Rückfall BREAKOUT_PENDING → CONSOLIDATING**
→ Implementiert: Aktuelles Bar's High/Low wird in Bounds eingearbeitet + Range-Revalidierung vor State-Revert.

**Bug 3 (Falsch positiv): SIGNAL_ACTIVE State-Lock**
→ Nicht geändert. `resetPatternState()` wird inline nach Signal-Emit aufgerufen, kein Lock.

### Dimension 2 — Konzept-Treue

Gemini bestätigt: 60-Bar RTH_OPENING Window ist korrekt (universell anerkannte Opening Range). Close-basiertes Breakout-Kriterium für 1-Minuten-Bars ist angemessen und reduziert Wick-Traps. Long-Only ist by design.

### Dimension 3 — Praktische Handelsfähigkeit

PatternSignal-Felder (Direction, Timestamp, Trigger Price): Bewusst nicht umgesetzt — separate Story. RulesEngine berechnet bereits 2:1 R:R aus entry/stop. Execution wird im OMS-Modul gehandhabt.

---

## 6. Pre-existing Issue

`IntentSignerVerifierIntegrationTest.java` — GELÖST. `mvn verify -pl odin-brain` läuft 11 Tests in diesem Test durch (0 Failures). Der zuvor gemeldete Issue war auf isolierten `-pl odin-brain` ohne `-am` zurückzuführen (odin-execution JAR nicht im lokalen Repository). Mit `mvn verify` wird die volle Modulkette gebaut.

---

## 7. QS-Agent Fixes (2026-02-22)

### Behobene Findings

| Finding | Fix |
|---------|-----|
| Fehlende `/** {@inheritDoc} */` auf 4 `@Override`-Methoden | Ergänzt in `OpeningConsolidationFsm.java` |
| Breakout-Bar Low nicht in `consolidationLow` | `consolidationLow = Math.min(consolidationLow, currentBar.low())` beim Übergang CONSOLIDATING→BREAKOUT_PENDING |
| `OPENING_WINDOW_BARS` JavaDoc unscharf | "Bars 1–60 are processed within the window; the 61st bar triggers expiry." |
| Kein Test "nach Timeout kein Signal" | Test `afterOpeningWindowExpired_perfectBreakoutAndVolume_doesNotEmitSignal` hinzugefügt |
| Kein Test "Breakout-Bar Wick unter consolidationLow" | Test `breakoutBarWithWickBelowConsolidationLow_stopLevelIncludesWick` hinzugefügt |

### Finales Testergebnis nach QS-Fixes

- Unit Tests (Surefire): **462/462 PASS** (29 in OpeningConsolidationFsmTest)
- Integration Tests (Failsafe): **29/29 PASS** (4 in OpeningConsolidationFsmIntegrationTest)
- BUILD SUCCESS

**QS R1: PASS** — 2026-02-22
