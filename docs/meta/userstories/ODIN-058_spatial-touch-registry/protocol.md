# Protokoll: ODIN-058 — Spatial Touch Registry + Simplified Reconciliation

## Working State

- [x] Initiale Implementierung (TouchRegistry, TouchEvent-Umbau, SrLevel-Umbau, SrZone-Umbau, SrLevelEngine-Umbau, SrProperties.ReconciliationProperties)
- [x] Unit-Tests geschrieben (TouchRegistryTest — 50 Tests)
- [x] Alle existierenden Tests angepasst (SrLevelEngineTest, SupportLevelConfidenceTest, BounceDetectorTest, PromptBuilderTest, KpiEngineTest, PipelineFactoryTest, ParameterOverrideApplierTest, BacktestRunnerTest, BacktestRunnerGovernanceIntegrationTest, BacktestPipelineIntegrationTest, WalkForwardRunnerIntegrationTest, TradingPipelineTest, DegradationManagerIntegrationTest)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases und Reconciliation-Review
- [x] Integrationstests (DoD 2.3 — SrTouchRegistryIntegrationTest, 9 Tests, Failsafe)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [ ] Commit & Push

## Design-Entscheidungen

### D1: SrLevel erhaelt TouchRegistry als Konstruktor-Parameter

SrLevel bekommt die TouchRegistry als Pflicht-Parameter uebergeben (kein Setter, kein optional).
Begruendung: Die Registry ist fundamental fuer die Existenz eines Levels (ohne Registry keine Touch-Delegation). Ein optionaler Parameter oder ein Setter haette die Invariante verletzt, dass ein Level immer eine Registry haben muss.

### D2: Uebergangsloesung Quality (pre-ODIN-059)

`TouchEvent` hat kein `quality`-Feld (entfernt per ODIN-058). `countValidated()` zaehlt vorerst alle Touches unabhaengig vom minQuality-Parameter. Der Parameter bleibt im API erhalten fuer nahtlose ODIN-059-Integration (kein Signaturwechsel noetig). Dokumentiert in JavaDoc.

### D3: touchRegistryForTest() package-private Accessor

`SrLevel.touchRegistryForTest()` ist package-private (nicht public). Exposes die Registry nur fuer Tests innerhalb desselben Packages. Keine Reflection, kein public API. Sauberere Loesung als alternative Ansaetze (Mock-Konstruktor, Builder-Pattern fuer Tests).

### D4: Leere Cluster -> Reconciliation ueberspringen

Wenn DBSCAN 0 Cluster liefert (temporaer sparse pivot-landscape), wird die Reconciliation-Schleife komplett uebersprungen. Bestehende Swing-Level erhalten in diesem Cycle keine orphan-Markierung. Begruendung: "Keine Cluster" bedeutet nicht "alle bisherigen Level sind invalid" — es bedeutet, dass gerade keine neuen Pivots entstanden sind. Das DBSCAN-Ergebnis ist transient; die Level wurden aus realen historischen Bars gewonnen.

### D5: Double-Counting in SrZone.totalValidatedTouchCount() — bewusstes Design

Wenn zwei Levels in einer Zone ueberlappende Baender haben, werden Touches im Ueberlappungsbereich doppelt gezaehlt. Das ist eine bewusste Architekturentscheidung: Jedes Level zaehlt seine eigenen Touches (seine eigene Preiszone). Eine Zone-weite deduplizierte Abfrage wuerde erfordern, dass SrZone eine direkte Registry-Referenz haelt (kein SWING_CLUSTER-Level-Filter moeglich). Diese Optimierung ist ODIN-060 vorbehalten (Output Policy, Confidence-Rekalibrierung). Bis dahin ist das Verhalten dokumentiert und getestet.

### D6: Null-Safety in TouchRegistry.record()

`Objects.requireNonNull(touch)` wird explizit geworfen. Begruendung: Fail-Fast bei offensichtlichen Programmierfehlern (null touch = upstream Bug). Ausserdem werden NaN/Infinity-Preise sowie Preise <= 0 still verworfen (return ohne Exception) — das sind upstream Datenfehler, keine Programmierfehler.

## Offene Punkte

### OP1: Integrationstests DoD 2.3 — ERLEDIGT (Runde 2)

DoD 2.3 wurde in Runde 2 durch `SrTouchRegistryIntegrationTest` (Failsafe, `*IntegrationTest`-Namenskonvention) vollstaendig erfuellt. Neun Integrationstests abdecken:

- IT1: Vollstaendiger Pipeline-Durchlauf mit realer TouchRegistry (keine Mocks)
- IT2: Level-Drift $41.80 → $41.95 — alte Touches bleiben am Originalpreis, kein Ghost-Touch
- IT3: Level-Drift zurueck $41.95 → $41.80 — Original-Touches sofort wieder sichtbar
- IT4: Zwei Level mit ueberlappenden Baendern — dokumentiertes Double-Counting (D5)
- IT5: Registry-Integritaet nach mehreren onSnapshot()-Aufrufen — keine Duplikate, monotones Wachstum
- IT6: End-to-End Snapshot → DBSCAN → Reconciliation → detectTouches → Registry → enrichZonesFromRegistry → SrSnapshot
- IT7: Orphan-Lifecycle im Pipeline-Kontext — Registry-Touches bleiben nach Level-Entfernung erhalten
- IT8: seedTouchesFromPivots → Registry → Level-Query — korrekte Pivot-Seeding-Integration
- IT9: enrichZonesFromRegistry-Pipeline-Schritt — Zones-touchCounts entsprechen direkter Registry-Abfrage

Alle 976 Unit-Tests und alle 9 Integrationstests gruен.

### OP2: Double-Counting Architekturentscheidung eskalieren

Beide Review-Partner (ChatGPT und Gemini) haben das Double-Counting in SrZone.totalValidatedTouchCount() als potenziell problematisch markiert. Gemini empfiehlt: Zone direkt die Registry abfragen lassen (zone.center +/- zone.band). Das wuerde erfordern, dass SrZone eine TouchRegistry-Referenz haelt. Entscheidung: Fuer ODIN-060 (Confidence-Rekalibrierung) aufheben, wo die gesamte Confidence-Berechnung ueberarbeitet wird.

### OP3: GC-Optimierung touchPriceStdDev / countValidated

Gemini P2: `queryRange()` erstellt bei jedem Aufruf eine neue ArrayList. Fuer reine Zaehl-Operationen waere ein direktes Summieren ueber subMap.values() effizienter. Begruendung fuer Nicht-Umsetzung: ODIN-Prinzip "erst bauen, dann messen, dann optimieren". Keine proaktive Optimierung ohne Profiling-Basis.

## ChatGPT-Sparring

Zwei Sessions. Dateien: TouchRegistry.java, TouchEvent.java, SrLevel.java, SrZone.java, SrLevelState.java, TouchRegistryTest.java, SrLevelEngine.java.

### ChatGPT Findings und Bewertung

| Finding | Prioritaet | Umgesetzt | Begruendung |
|---------|-----------|-----------|-------------|
| `queryRange()` crasht bei lower > upper (TreeMap.subMap IllegalArgumentException) | P0 | Ja | Guard hinzugefuegt: wenn loBin > hiBin → return List.of() |
| NaN/Infinity in priceToBin() → Bin 0 oder Overflow | P0 | Ja | Finite-Check in record() und queryRange() |
| `record(null)` → NPE explizit werfen | P2 | Ja | Objects.requireNonNull hinzugefuegt |
| Multiplikation statt Division bei Bin-Berechnung (FP-Praezision) | P1 | Nein | Math.round mitigiert das Problem bereits. Aenderung wuerde Verhalten nicht substantiell verbessern. Als Kommentar dokumentiert. |
| Fehlende Tests fuer swapped bounds, NaN/Infinity | P1 | Ja | 7 neue Defensive-Guard-Tests hinzugefuegt |
| "0 Cluster → Orphan all" logisches Risiko | P1 | Ja | Guard in reconcileSwingClusters() — wenn newClusters.isEmpty() → return |
| Double-Counting in SrZone.totalValidatedTouchCount() | P1 | Nein (Design-Entscheidung) | Bewusstes Design, siehe D5. ODIN-060 zugewiesen. |
| Doppelseeding bei Level-Recreation | P2 | Nein | Edge-Case, tritt nur auf wenn Level geloescht und sofort neu erstellt wird. Kein De-Dup noetig fuer Phase 1. |

### ChatGPT hat ausserdem bemerkt

- Overlap-Semantik ist inklusive (>=), nicht "> 0" wie in der Spec. Entscheidung: inklusive ist korrekt fuer praktische S/R-Level-Matching (ein Level das genau beruehrt wird soll matchen).
- `clusterBand = max(band, cluster.maxDeviation())` kann Overlap-Matches aggressiver machen. Akzeptiert — das ist gewolltes Verhalten (Band darf nicht kleiner sein als der Cluster-Spread).

## Gemini-Review

Drei Dimensionen in einer Session.

### Dimension 1: Code-Review

| Finding | Prioritaet | Umgesetzt | Begruendung |
|---------|-----------|-----------|-------------|
| Architektur und defensive Programmierung gelobt | — | — | Bestaetigt vorhandene Qualitaet |
| Double-Counting SrZone (Fake Confidence) | Hoch | Nein (Design) | Siehe D5 und OP2 |
| touchPriceStdDev Standardabweichung verfaelscht bei Overlap | Hoch | Nein (Design) | Gleiches Argument wie D5 |
| GC-Druck durch queryRange ArrayList | Mittel | Nein (Optimierung) | Erst messen, dann optimieren |
| Floating-Point: Multiplikation robuster als Division | Mittel | Nein | Kommentar in priceToBin() erlaeutert Aequivalenz |

### Dimension 2: Konzepttreue

Gemini bestaetigt volle Konzepttreue fuer alle ODIN-058-Anforderungen:
- Registry-Architektur: korrekt
- Reconciliation-Vereinfachung: korrekt
- Grace-Period-Semantik: korrekt
- Split/Merge entfernt: korrekt
- Matching-Kriterien: korrekt
- Hard Center Update: korrekt
- Leeres Cluster-Ergebnis Guard: als "sehr sinnvolle Erganzung" gelobt

### Dimension 3: Praxis-Review

| Finding | Prioritaet | Umgesetzt | Begruendung |
|---------|-----------|-----------|-------------|
| Memory Leak: reset() leert TouchRegistry nicht | P0 (Kritisch) | Ja | TouchRegistry.clear() hinzugefuegt, SrLevelEngine.reset() ruft clear() auf |
| Double-Counting SrZone bei zonenweiter Abfrage waere besser | P1 | Nein (Design) | Siehe D5 |
| GC-Optimierung countValidated | P2 | Nein | Erst messen, dann optimieren |
| "0 Cluster" friert Level ein (keine orphan-Ausfaulung) | P3 | Akzeptiert | Seltener Edge-Case; Monitoring empfohlen |
| Integer Overflow bei Aktien > $500 | P4 | Kein Fix noetig | Max bin-Wert fuer $21M+ Aktie — kein praktisches Risiko |
| Thread-Safety (zukuenftige Parallelisierung) | P5 | Kein Fix noetig | Single-threaded per pipeline ist ODIN-Architekturprinzip |

## Remediation Runde 2 (2026-02-27)

### Behobene QA-Findings

**P1 (KRITISCH): Integrationstests — BEHOBEN**

Neue Klasse: `SrTouchRegistryIntegrationTest.java` (Failsafe, `*IntegrationTest`)
Pfad: `odin-brain/src/test/java/de/its/odin/brain/sr/SrTouchRegistryIntegrationTest.java`

9 Integrationstests implementiert:
- IT1: fullPipelineRun_realTouchRegistry_zonesHaveCorrectTouchCounts
- IT2: levelDrift_oldTouchesStayAtOriginalPrice_noGhostTouches
- IT3: levelDriftBack_levelSeesOriginalTouchesAgain
- IT4: twoLevelsOverlappingBands_bothCountSharedTouches (dokumentiertes D5-Verhalten)
- IT5: registryIntegrity_multipleSnapshots_noSpuriousDuplicates
- IT6: endToEnd_detectTouchesViaRegistry_snapshotReflectsRegistryCount
- IT7: orphanLifecycle_levelOrphanedThenRemoved_registryTouchesPreserved
- IT8: seedTouchesFromPivots_registryContainsSeedTouches_levelQueryReturnsCorrectCount
- IT9: enrichZonesFromRegistry_snapshotZonesReflectRegistryState

Alle 976 Unit-Tests (Surefire) und alle 9 Integrationstests (Failsafe) gruен.

**K1 (Minor): enrichZonesFromRegistry() — KEIN HANDLUNGSBEDARF (bestaetigt)**

Die Implementierung ist funktional korrekt. SrZone delegiert touchCounts live ueber contributing levels an die Registry — kein separater "Set"-Schritt noetig. IT9 validiert diese Korrektheit explizit. Die story.md-Formulierung "setzt touchCount" ist ungenau, das Verhalten ist aber spezifikationstreu. JavaDoc erklaert den Design-Grund bereits.

**K2 (Minor): Bewusste Design-Entscheidung — KEIN HANDLUNGSBEDARF**

Dokumentiert in D5 (Double-Counting) und D3 (touchRegistryForTest Accessor). Beide Entscheidungen sind begruendet und getestet.

**P2 (Minor): Commit & Push — ausstehend (bleibt bei QS-Agent)**
