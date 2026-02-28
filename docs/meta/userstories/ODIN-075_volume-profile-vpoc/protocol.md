# Protokoll: ODIN-075 -- Volume Profile (VPOC/VAH/VAL)

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (26 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases (9 zusaetzliche Tests)
- [x] Integrationstests geschrieben (4 Tests)
- [x] Gemini-Review Dimension 1 (Bugs)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### Bin-Algorithmus
- Uniform volume distribution: Bar-Volume wird gleichmaessig auf alle Bins zwischen Low und High verteilt
- Bin-Index via `Math.floor(price / binSize)` -- deterministisch, effizient
- VPOC = Mid-Price des Bins mit hoechstem kumulativem Volumen
- VAH = obere Kante des obersten Value-Area-Bins
- VAL = untere Kante des untersten Value-Area-Bins

### Value Area Expansion
- Single-Bin-Expansion: abwechselnd oben/unten, der Bin mit mehr Volumen gewinnt
- Gemini-Review notierte dass TPO-Standard dual-bin (2 Bins gleichzeitig pruefen) nutzt
- Entscheidung: single-bin ist mathematisch valide und ausreichend fuer Intraday

### S/R-Level-Integration
- VP-Levels sind DYNAMISCH (wie VWAP) -- Center wird bei jedem neuen Bar aktualisiert
- VP_POC Prior: 0.70 (institutioneller Referenzwert, hoeher als VAH/VAL)
- VP_VAH/VP_VAL Prior: 0.60
- SrLevelEngine hat ueberladenen Konstruktor: ohne VP-Calculator = keine VP-Levels

### Inkrementelle Berechnung
- `volumeProfileBarsFed` Counter verhindert doppeltes Feeding
- addBar() akkumuliert in Bin-Map, compute() berechnet aus Bin-Map
- O(bins) pro Update, nicht O(bars * bins)

### Konfiguration
- Namespace: `odin.brain.kpi.volume-profile.*`
- VolumeProfileProperties Record in BrainProperties.KpiProperties
- Defaults: binSize=0.05, valueAreaPercent=70.0, vpocInitialConfidence=0.70

## ChatGPT-Sparring

Ergebnis: 9 zusaetzliche Edge-Case-Tests implementiert:

1. `compute_orderIndependence` -- Reihenfolge der Bars aendert VPOC nicht
2. `compute_vpocAtEdge_lowestBin` -- VPOC korrekt wenn am unteren Rand
3. `compute_vpocAtEdge_highestBin` -- VPOC korrekt wenn am oberen Rand
4. `compute_sparseBinsWithGaps` -- Luecken zwischen Bins korrekt behandelt
5. `invariant_valLessEqualVpocLessEqualVah` -- VAL <= VPOC <= VAH immer
6. `compute_smallValueAreaPercent` -- 10% Value Area korrekt
7. `compute_duplicateBars_equivalentToSummedVolume` -- Duplikate = Summe
8. `compute_tinyPrices_subPenny` -- Sub-Penny-Preise funktionieren
9. `compute_allZeroVolume_returnsEmpty` -- Nur Null-Volumen = leer

## Gemini-Review

### Dimension 1 -- Bugs

**Gefunden:**
1. **Non-deterministische VPOC Tie-Breaking** -- HashMap-Iteration ist nicht determiniert. Bei exakt gleichem Volumen UND gleichem Abstand zum Median war das Ergebnis zufaellig.
   - **FIX angewendet:** Zusaetzlicher Tie-Breaker `bin < bestBin` in findVpocBin()
2. **Integer Overflow in priceToBin()** -- Bei extremen Preisen ($600k) und kleinem binSize koennte int ueberlaufen.
   - **DEFER:** ODIN handelt US-Equities im Bereich $1-$500. BRK.A ist kein Zielinstrument.
3. **Global Boundary Edge Case** -- globalMinBin/globalMaxBin bei leerer Map.
   - **Mitigiert:** Early-Return Guard faengt diesen Fall ab.

### Dimension 2 -- Konzepttreue

- VPOC-Definition: Korrekt
- VAH/VAL-Kanten: Korrekt
- Value Area Expansion: Valide (single-bin statt dual-bin)
- Volume Distribution: Standard-Approximation fuer Candlestick-Daten

### Dimension 3 -- Praxis

1. **Performance bei hohen Preisen** -- Statischer binSize problematisch bei grosser Preisspanne.
   - **DEFER:** binSize ist konfigurierbar. Dynamisches Sizing (ATR-basiert) als Future Enhancement.
2. **Penny Stock Praezision** -- $0.05 binSize bei $0.02 Aktie = nur 1 Bin.
   - **DEFER:** Kein Penny-Stock-Trading geplant.
3. **HashMap Object Overhead** -- Boxing von Integer/Double.
   - **DISCARD:** Premature Optimization. Typischerweise <500 Bins, GC-Druck vernachlaessigbar.
4. **Integration Robustness** -- Null-Handling und inkrementelles Tracking korrekt.

## Offene Punkte

Keine offenen Punkte. Alle Findings bewertet und dokumentiert.

## Dateien

### Neue Dateien
- `odin-brain/src/main/java/de/its/odin/brain/kpi/VolumeProfile.java`
- `odin-brain/src/main/java/de/its/odin/brain/kpi/VolumeProfileCalculator.java`
- `odin-brain/src/test/java/de/its/odin/brain/kpi/VolumeProfileCalculatorTest.java`
- `odin-brain/src/test/java/de/its/odin/brain/sr/VolumeProfileSrLevelIntegrationTest.java`

### Geaenderte Dateien
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` (VolumeProfileProperties)
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelSource.java` (VP_POC prior 0.60->0.70)
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelEngine.java` (VP integration)
- `odin-brain/src/main/resources/odin-brain.properties` (VP config)
- `odin-brain/src/test/java/de/its/odin/brain/TestBrainProperties.java`
- 8 weitere Testdateien (KpiProperties-Konstruktor-Kompatibilitaet)

## Testergebnis

- VolumeProfileCalculatorTest: 26/26 pass
- VolumeProfileSrLevelIntegrationTest: 4/4 pass
- odin-brain gesamt: 1356 Tests, 0 Failures
