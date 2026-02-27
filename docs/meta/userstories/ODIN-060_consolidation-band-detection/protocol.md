# Protokoll: ODIN-060 — Consolidation Band Detection

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (17 Tests, alle grün)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (3 Tests, alle grün)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [ ] Commit & Push

## Design-Entscheidungen

### D1: DenseRegion als immutable Record
DenseRegion wurde als immutable Record mit einer `extendTo(newTop)` Methode implementiert die eine neue Instanz
zurückgibt. Das Konzept-Pseudocode hatte eine mutable `current.extendTo(...)` API — das wurde beibehalten (durch
Neuzuweisung: `current = current.extendTo(upper)`) weil immutable Records sicherer gegen unbeabsichtigte Mutation sind.

### D2: POC-Berechnung mit volumeRatio als Quality-Proxy
Pre-ODIN-059 kennt TouchEvent keine explizite Quality-Zahl. Als Proxy wird `volumeRatio` verwendet (wie in der
`TouchQuality`-Klasse bereits als Teilkomponente genutzt). Nach ODIN-059-Integration kann auf dynamische Quality
umgestellt werden. Das ist korrekt da das Konzept sagt: "Quality als Gewicht" — und volumeRatio ist die am besten
verfügbare Single-Zahl in der Registry.

### D3: Monotonisches Wachstum über previousBands-Liste
Da `ConsolidationBandDetector` ein POJO per Pipeline ist, kann er previous state zwischen Snapshots halten.
Der overlap-check identifiziert "dasselbe Band" durch Preis-Overlap (nicht durch eine ID). Das ist korrekt weil
Bands stabil sind und sich nur in Grenzbereichen überlappen können.

**Fix nach Review:** `applyMonotonicGrowth` merged nun alle überlappenden previousBands (nicht nur das erste),
um korrekt mit dem Fall umzugehen dass ein neu erkanntes Band zwei bisher separate Bänder brückt.
**Fix nach ChatGPT-Review:** `previousBands` wird nur geupdated wenn `detected` nicht leer ist. Wenn ein
Snapshot transient unter `minTouchesPerWindow` fällt, bleibt die Band-History erhalten — monotonisches Wachstum
setzt beim nächsten Snapshot fort.

### D4: Cluster-Ersetzung nur wenn mindestens 1 SWING_CLUSTER-Zone
Laut Spec: "Wenn ein Band 0 Cluster-Linien ersetzt, wird kein Band erzeugt." Das ist implementiert in
`replaceClusterZonesWithBands()`. Wenn nur statische Level im Band-Bereich liegen, wird das Band übersprungen.

### D5: CONSOLIDATION als neuer SrLevelSource-Enum-Wert
Statt String-Vergleich wird ein echter Enum-Wert eingeführt. Prior=0.70 (zwischen VWAP=0.85 und SWING_CLUSTER=0.60),
da eine Konsolidierungszone mehr strukturelle Relevanz als ein einzelner Cluster hat aber weniger als VWAP.
Category="CONSOLIDATION" für saubere Confluence-Counting-Isolation.

### D6: Band-Zone bekommt leere TouchRegistry
Die ConsolidationBand-Zone nutzt eine leere neue TouchRegistry im synthetischen SrLevel. Das ist korrekt weil
die Band-Touches bereits in der Haupt-Registry gespeichert sind und für die Band-Zone keine direkte Level-Touch-
Delegation gebraucht wird. Confidence kommt aus `avgTouchQuality` des Bands.

### D7: Scan-Window-Clamping auf [scanLow, scanHigh]
Die Sliding-Window-Grenzen `lower/upper` werden auf `[scanLow, scanHigh]` geclampt um sicherzustellen dass keine
Touches außerhalb des deklarierten Scan-Bereichs gezählt werden. Konsequenz: Rand-Fenster sind schmaler als W und
erfordern denselben absoluten Schwellenwert — Rand-Bereiche werden damit natürlich benachteiligt. Das ist
akzeptabel und semantisch korrekt (Scan-Range ist ein harter Bound).

### D8: avgTouchQuality kann > 1.0 sein
`computeAvgQuality()` mittelt `volumeRatio` (Proxy für Quality) ungeclampt. volumeRatio ist ein relatives Maß
(kein normierter Wert auf [0,1]). Die resultierende `avgTouchQuality` im Band kann > 1.0 sein und wird als
Zone-Confidence gesetzt. Das ist bewusste Design-Entscheidung: Downstream-Code muss robust mit Werten >1.0 umgehen.
Nach ODIN-059-Integration wird Quality auf [0,1] normiert sein.

## Offene Punkte

*Aus Gemini Dimension 3 — eskaliert an Stakeholder (nicht eigenständig gelöst):*

### OP-1: Gap Tolerance innerhalb großer Konsolidierungsbänder
Wenn innerhalb eines perfekten Chop-Bands kurzzeitig eine Lücke entsteht (Preis pendelt schnell von Top zu Bottom),
kann ein einzelnes Sliding-Window unter `minTouchesPerWindow` fallen. Folge: ein Band wird zu zwei separaten
DenseRegions. Das monotone Wachstum hilft teilweise, aber wenn beide Regionen noch nicht in previousBands sind,
entstehen zwei Bänder statt eines.

*Implikation:* Eine "Gap Tolerance" (z.B. max 1 leeres Fenster innerhalb einer DenseRegion akzeptieren) könnte
die Stabilität verbessern. Nicht Teil von ODIN-060, Kandidat für ODIN-06x.

### OP-2: Dynamische minTouchesPerWindow basierend auf Session-Aktivität
Ein starrer Default von 5 Touches/Fenster ist für Mid-Session bei High-Beta-Aktien gut. Für sehr liquide
Instrumente (SPY, TSLA) könnte nahezu der gesamte Chart als Band erkannt werden. Für illiquide Small Caps
werden Bänder nie erkannt. Eine relative Dichte (% der Session-Touches im Fenster) wäre robuster.

*Implikation:* Nur relevant wenn ODIN multi-instrument mit stark unterschiedlicher Liquidität arbeitet.
Derzeit kein unmittelbarer Handlungsbedarf — beobachten im Betrieb.

### OP-3: Lebenszyklus veralteter Bands (Time-to-Live)
Bands wachsen monoton bis EOD (`reset()`). Ein Band das am Vormittag bei $40 gebildet wurde und der Preis
nachmittags bei $50 steht, wird weiterhin exportiert. Das kann Downstream-Engines mit irrelevanten alten
Zonen belasten. Das Konzept behandelt diesen Fall nicht.

*Implikation:* TTL/Decay für Bänder die seit N Snapshots nicht mehr berührt wurden. Kandidat für ODIN-06x
oder als Teil des Output-Policy-Updates (ODIN-062).

## ChatGPT-Sparring

**Session:** 2 Runden, 2026-02-27

**Vorgeschlagene Edge Cases:**

| # | Vorschlag | Bewertung | Aktion |
|---|-----------|-----------|--------|
| 1 | Scan-Boundary overshoot (lower/upper beyond scanLow/scanHigh) | Bestätigt als Bug | Behoben: clamp lower/upper auf [scanLow, scanHigh] |
| 2 | Single-Price False Positive (alle Touches exakt gleicher Preis) | Partiell valid, minSpread fängt Extremfälle ab | Testfall vorhanden (minSpread-Test), kein weiterer Fix |
| 3 | validated vs. all Touches für Band-Stats | Design Decision: queryRange liefert alle Touches bewusst | Dokumentiert in D2, kein Fix |
| 4 | avgTouchQuality > 1.0 | Bestätigt, bewusste Design-Entscheidung | Dokumentiert in D8 |
| 5 | State-Loss bei leerer Detection (monotonic growth History) | **Kritischer Bug bestätigt** | Behoben: previousBands nur updaten wenn detected nicht leer |
| 6 | Overlapping Bands / Split-Merge Instabilität | Partiell, applyMonotonicGrowth merged nur erstes Band | Teilweise behoben durch Multi-Band-Merge |
| 7 | Tests mit silent skip auf bands.isEmpty() | Bestätigt als Test-Weakness | Behoben: 4 Tests auf assertFalse umgestellt |

**Neue Tests aus ChatGPT-Review:**
- Test 17: `monotonicGrowth_emptyDetectionBetweenSnapshots_historyRetained` — verifiziert den State-Loss-Fix

**Verworfene Vorschläge:**
- "Konfig-Guard maxWindows": Nicht notwendig, Performance ist bei ±3 ATR unkritisch (~64 Schritte)
- "Epsilon-Vergleiche für Floating-Point": Nicht notwendig, doppelte Grenzen (<=/>= statt strict) sind robust genug für Preis-Granularität von $0.01

## Gemini-Review

**Session:** 1 Runde, 2026-02-27

### Dimension 1: Code-Review Findings

| # | Finding | Bewertung | Aktion |
|---|---------|-----------|--------|
| 1 | State Wipe: previousBands wird auch bei nicht-leerer neuer Erkennung zurückgesetzt | Teilweise bereits behoben (ChatGPT-Fix), Gemini erweitert: auch bei Wechsel zu neuem Band | Analysiert: kein weiterer Fix nötig — neues Band ersetzt korrekterweise das alte |
| 2 | Half-Window at Boundary: Rand-Fenster sind schmaler als W | Valid, aber akzeptabel — Rand-Bereiche werden natürlich benachteiligt | Dokumentiert in D7, kein Code-Fix |
| 3 | Multi-Band Merge: applyMonotonicGrowth merged nur mit erstem überlappenden previousBand | **Echter Bug** | Behoben: Loop über alle previousBands, akumuliert min/max |

### Dimension 2: Konzepttreue-Review

Gemini bestätigt hohe Konzepttreue:
- ConsolidationBand-Record-Felder: 1:1 mit Spec
- POC-Formel: korrekt adaptiert (volumeRatio als Proxy ist begründet)
- Zonen-Austausch: SWING_CLUSTER raus, statische Level bleiben — korrekt
- Pipeline-Reihenfolge: korrekt (nach NMS, vor Output Policy)

**Abweichung:** Loop-Iteration über Center statt Preis direkt — Gemini akzeptiert das als äquivalent.

### Dimension 3: Praxis-Review (Offene Punkte)

Alle drei Punkte wurden als OP-1, OP-2, OP-3 unter "Offene Punkte" dokumentiert und an Stakeholder eskaliert.
