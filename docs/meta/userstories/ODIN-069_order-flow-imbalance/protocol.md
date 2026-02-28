# Protokoll: ODIN-069 — Order Flow Imbalance (OFI)

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (27 OfiCalculatorTest + 16 OfiGateTest = 43 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (GateCascadeEvaluatorTest + KpiEngineTest abgedeckt)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen
- OFI-Berechnung folgt Cont/Kukanov/Stoikov 2014 exakt: deltaBid basiert auf bidPrice-Veraenderung (Preis hoch = +bidSize, Preis runter = -bidSize(t-1), Preis gleich = delta-Size)
- Z-Score Normalisierung mit rollierendem Fenster ueber die letzten N OFI-Werte (Default 20), implementiert als Circular-Buffer mit Running-Sum/Running-SumSquared
- OfiGate als Gate #2 NACH SpreadGate in der Kaskade (Spread ist billiger zu berechnen), VOR VolumeGate
- GateType.OFI als zweiter Enum-Wert (normative Ordnung: SPREAD, OFI, VOLUME, RSI, EMA_TREND, VWAP, ATR, ADX)
- OFI ist Default-off (Feature-Flag odin.brain.ofi.enabled=false), Gate gibt bei disabled immer PASS zurueck
- Bei fehlenden Top-of-Book-Daten (NaN): Gate gibt PASS mit Warning zurueck (fail-open Design)
- Z-Score Warmup: Erste N Bars (=z-score-window) haben keinen validen Z-Score, Gate gibt PASS zurueck
- MarketSnapshot wird um bestBidPrice, bestAskPrice, bestBidSize, bestAskSize erweitert (NaN Default)
- TopOfBookSnapshot als separater Record in odin-api fuer strukturierte Weitergabe
- Incomplete Snapshots werden uebersprungen ohne previousSnapshot zu korrumpieren — sicherer State-Erhalt
- MIN_STD_DEV Guard (1e-10) gegen Division durch Null bei identischen OFI-Werten
- Negative Varianz durch Floating-Point-Fehler wird auf 0.0 geclampt

## QA Round 1 — Findings & Remediation (R2)

### QA-Report
Datei: `its-odin-backend/temp/userstories/ODIN-069/qa-report-r1.md`
Ergebnis: FAIL (4 Findings: 2 CRITICAL, 1 SEVERE, 1 MINOR)

### M1 (CRITICAL) — Fehlende IntegrationTest-Klassen
**Problem:** Nur `*Test`-Klassen vorhanden (Surefire), keine `*IntegrationTest`-Klassen (Failsafe).
**Fix:** Zwei neue IntegrationTest-Klassen erstellt:
- `GateCascadeEvaluatorIntegrationTest.java` (odin-brain): 9 Tests — vollstaendige OFI Gate-Cascade E2E mit realen Gate-Implementierungen (keine Mocks). Testet OFI disabled/enabled, pass/fail, elevated mode, NaN handling, short-circuit.
- `KpiEngineIntegrationTest.java` (odin-brain): 6 Tests — KpiEngine + OfiCalculator E2E. Testet no-ToB=NaN, first-valid-ToB=baseline-only, two-valid-ToB=ofiRaw, enough-observations=ofiZScore, reset, incomplete-ToB.

### M2 (CRITICAL) — AC#6 IB Broker ToB-Extraktion
**Problem:** `OdinEWrapper`/`IbMarketDataFeed` extrahierten keine Top-of-Book-Daten aus tickPrice/tickSize Callbacks.
**Fix (3 Dateien):**
- `OdinEWrapper.java`: ToB-State-Tracking per reqId via ConcurrentHashMap. tickPrice() speichert BID/ASK-Preise (Field 1/2), tickSize() speichert BID/ASK-Sizes (Field 0/3). Cleanup in removeTickListener(). Neue Methode getTopOfBookState().
- `EventPayloadParser.java`: TickData Record um optionale bidSize/askSize erweitert (NaN wenn nicht im Payload). Neue extractOptionalDouble() Methode.
- `DataPipelineService.java`: bestBidSize/bestAskSize Felder hinzugefuegt, processTick() extrahiert Sizes aus TickData, createCurrentSnapshot() uebergibt currentBid/currentAsk (mit >0 Guard) und tracked Sizes statt NaN.

### M3 (SEVERE) — FULL_GATE_COUNT nicht aktualisiert
**Problem:** `GateCascadeIntegrationTest.FULL_GATE_COUNT = 7` statt 8 nach OFI-Integration.
**Fix:** Konstante auf 8 aktualisiert, alle Test-Evaluierungslisten um GateType.OFI erweitert.

### M4 (MINOR) — GateType.OFI nicht in Integrationstest-Szenarien
**Problem:** OFI fehlte in allen Szenarien des bestehenden Integrationstests.
**Fix:** GateType.OFI als zweiter Eintrag (nach SPREAD) in allen Evaluierungslisten hinzugefuegt. Neuer Test `fullCascade_ofiGateFails_firstFailedGateIsOfi()` erstellt.

### Test-Ergebnisse R2
| Modul | Test-Klasse | Tests | Status |
|-------|------------|-------|--------|
| odin-api | GateCascadeIntegrationTest | 4 | PASS |
| odin-brain | OfiCalculatorTest | 27 | PASS |
| odin-brain | OfiGateTest | 16 | PASS |
| odin-brain | GateCascadeEvaluatorTest | 44 | PASS |
| odin-brain | GateCascadeEvaluatorIntegrationTest | 9 | PASS |
| odin-brain | KpiEngineIntegrationTest | 6 | PASS |
| odin-data | EventPayloadParserTest | 6 | PASS |
| **Gesamt** | | **112** | **PASS** |

Build: `mvn clean install -DskipTests` — BUILD SUCCESS (alle 11 Module).

## Offene Punkte
- **Numerische Stabilitaet (Gemini Dim 3):** Naive Varianz-Formel (sumSquared/n - mean^2) koennte bei sehr grossen Werten numerische Instabilitaet zeigen. Welford's Online-Algorithmus waere praeziser, aber fuer die aktuellen OFI-Wertegroessen (max ~10^4) ist die naive Formel ausreichend. Bei Bedarf als Improvement implementieren.
- **Observation-Frequenz (Gemini Dim 3):** Das Z-Score-Fenster (Default 20) zaehlt OFI-Observationen, nicht Zeitfenster. Bei unterschiedlicher Tick-Frequenz (z.B. ruhige vs. volatile Phasen) koennte die Z-Score-Interpretation variieren. Fuer V1 akzeptabel, in V2 ggf. zeitbasiertes Fenster.

## ChatGPT-Sparring

### Session-Inhalt
ChatGPT erhielt: OfiCalculator.java, OfiGate.java, TopOfBookSnapshot.java, OfiCalculatorTest.java (20 Tests), OfiGateTest.java (11 Tests) + Akzeptanzkriterien.

### Vorgeschlagene Edge Cases (22 Szenarien)

**Umgesetzt (hoechste Prioritaet):**
- #8-9: Invalid Snapshot darf previousSnapshot nicht korrumpieren — `incompleteSnapshotBetweenValidOnes_doesNotCorruptState`, `multipleIncompleteSnapshots_doNotAdvanceBaseline`
- #12-14: Circular-Buffer-Wraparound mit deterministischer Z-Score-Berechnung — `circularBuffer_oldestValueDropped_deterministicZScore`, `circularBuffer_fullRotation_correctZScore`
- Disabled Gate NaN-Felder — `evaluate_disabled_actualValueAndThresholdAreNaN`
- Extreme Z-Score-Werte — `evaluate_extremeNegativeZScore_fails`
- Elevated Mode exakt am Threshold — `evaluate_elevated_exactlyAtElevatedThreshold_fails`
- NaN Z-Score in Elevated Mode — `evaluate_elevated_nanZScore_passesWithWarning`
- Erste Snapshot incomplete, zweite wird Baseline — `firstSnapshotIncomplete_secondBecomesBaseline`
- Near-Zero-Varianz Guard — `zScore_nearZeroVariance_returnsZeroNotNaN`
- Alternierende Werte fuer Running-Sum-Genauigkeit — `zScore_alternatingValues_correctSign`

**Bewusst nicht umgesetzt (mit Begruendung):**
- #1-4 (Zero Size, Negative Size): TopOfBookSnapshot validiert aktuell keine Werte — das ist bewusstes Design (isComplete prueft nur NaN). Negative/Zero-Sizes sind fachlich moeglich (Bid/Ask-Size = 0 nach Cancel).
- #5-7 (Crossed Market, Bid > Ask): Crossed-Market-Erkennung ist Aufgabe des Spread-Gates, nicht des OFI-Gates.
- #15-17 (Thread Safety): OfiCalculator ist per Design single-threaded (pro Pipeline), kein Concurrent-Test noetig.
- #18-22 (toString, hashCode, equals): Record-basierte Klassen haben diese automatisch, kein Test noetig.

## Gemini-Review

### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
**Ergebnis:** Keine kritischen Bugs gefunden. Code ist korrekt implementiert.
- Numerische Stabilitaet: Naive Varianz-Formel akzeptabel fuer erwartete Wertegroessen, Welford's als Improvement notiert
- NaN-Handling: Korrekt durchgaengig implementiert
- State-Management: previousSnapshot wird bei Incomplete korrekt beibehalten
- Circular Buffer: Korrekte Implementierung mit Running-Sum-Accounting

### Dimension 2: Konzepttreue-Review
**Ergebnis:** Implementierung folgt Cont/Kukanov/Stoikov 2014 korrekt.
- deltaBid/deltaAsk-Berechnung entspricht exakt dem Paper
- Vorzeichen-Konvention stimmt (positiv = Kaufdruck, negativ = Verkaufsdruck)
- Z-Score-Normalisierung ist korrekt (rolling mean und rolling stddev)
- Gate-Schwelle prueft korrekt auf starken Verkaufsdruck (Z-Score < threshold = FAIL)

### Dimension 3: Praxis-Review
**Findings:**
1. **Observation-basiertes vs. zeitbasiertes Fenster:** Z-Score-Fenster zaehlt Observationen, nicht Zeitfenster. Bei stark variierender Tick-Frequenz koennte dies zu unterschiedlicher Z-Score-Interpretierbarkeit fuehren. Fuer V1 akzeptabel.
2. **Fail-Open-Risiko:** Bei fehlenden Daten (NaN) laesst das Gate Entries durch. In der Praxis bedeutet das, dass bei einem IB-Datenausfall das OFI-Gate keine Schutzwirkung hat. Akzeptabel, da andere Gates (Spread, Volume) weiterhin aktiv sind.
3. **IB TWS Tick-Frequenz:** Best-Bid/Ask-Updates kommen in der Praxis ~250ms getaktet. Bei 1-Minuten-Bars werden also ~240 OFI-Updates pro Bar aggregiert. Der aktuelle 1:1-Ansatz (ein OFI pro Bar) ist korrekt fuer das Bar-basierte System.

### Bewertung
Alle Findings sind als "akzeptabel fuer V1" eingestuft. Keine erfordert sofortige Aenderung. Welford's Algorithmus und zeitbasiertes Fenster als moegliche V2-Improvements notiert.
