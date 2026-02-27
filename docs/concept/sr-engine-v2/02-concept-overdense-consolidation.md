# Mini-Konzept 2: Overdense Konsolidierungszone

**Problem:** 4 Cluster-Zonen in $0.37 Spanne (0.9%), NMS-Radius zu klein (0.10 vs 0.11–0.14 Abstände).
**Quellen:** ChatGPT-Sparring (1D-Scan + Kompression) + Gemini-Review (Flickering-Kritik, Band-Objekte, Registry-basierte Dichteerkennung)

---

## Gemini-Kritik am ursprünglichen ChatGPT-Konzept

Das ursprüngliche Konzept (1D-Scan → Gruppen → 3 Repräsentanten-Linien) hatte drei
strukturelle Schwächen:

**1. Level-Flickering (State Instability):**
> Ein neuer Touch ändert die ATR leicht. Der 1D-Scan ordnet Gruppen neu. Das
> Lower-Edge springt von 41.70 auf 41.82. Orders die gegen das Lower-Edge platziert
> wurden hängen im luftleeren Raum. Das ist dasselbe Identitäts-Problem wie bei
> Ghost Touches (Konzept 4).

**2. groupSize >= 3 ist falsch:**
> Zwei massive Level ($0.12 apart, je 50 Touches) werden NICHT als Konsolidierung
> erkannt weil groupSize nur 2. Die Relevanz einer Zone definiert sich über
> Touch-Dichte und Zeit, nicht über die Anzahl der NMS-Überlebenden.

**3. Konsolidierungen sind Zonen, nicht Linien:**
> 3 reparierte Linien verleiten Downstream-Engines (LLM, Rules) dazu, an der
> Lower-Line zu kaufen und an der Upper-Line zu shorten. In Wirklichkeit ist der
> gesamte Bereich "Chop". Das richtige Export-Format ist ein Band-Objekt.

---

## Neuer Lösungsansatz: Dichte-basierte Konsolidierungserkennung über Spatial Touch Registry

### Kernprinzip

**Konsolidierungszonen werden aus der Touch-Dichte erkannt, nicht aus der Anzahl
überlebender NMS-Peaks.**

Die Spatial Touch Registry (Konzept 4) speichert alle Touches räumlich indiziert.
Statt nachträglich NMS-Überlebende zu gruppieren, erkennen wir Konsolidierungszonen
direkt als zusammenhängende Bereiche hoher Touch-Dichte. Das Ergebnis ist ein
**Band-Objekt** (ConsolidationBand) statt einzelner Linien.

### Architektur: ConsolidationBand (neuer Zone-Typ)

```java
/**
 * A consolidation band represents a price range with high touch density
 * where price has oscillated without a clear directional break.
 * Unlike discrete S/R levels, a band signals "chop zone — trade the breakout,
 * not the interior".
 */
public record ConsolidationBand(
    double bottom,           // untere Grenze der Zone
    double top,              // obere Grenze der Zone
    double poc,              // Point of Control (touch-gewichteter Schwerpunkt)
    int totalTouches,        // kumulierte Touches im Band
    double avgTouchQuality,  // durchschnittliche Quality aller Touches
    Instant firstTouch,      // Zeitpunkt des ersten Touch (für Recency)
    Instant lastTouch        // Zeitpunkt des letzten Touch
) {}
```

### Erkennung: Touch-Dichte-Scan auf der Registry

Nach dem regulären S/R-Level-Scoring, vor der finalen Ausgabe:

```java
/**
 * Scans the touch registry for consolidation zones: contiguous price ranges
 * with high touch density and significant vertical spread.
 *
 * Algorithm:
 * 1. Slide a window of width W across the price range
 * 2. Count validated touches in each window position
 * 3. Identify contiguous regions where touch density exceeds threshold
 * 4. If region spread exceeds minSpread → ConsolidationBand
 */
private List<ConsolidationBand> detectConsolidationBands(
        TouchRegistry registry, double currentPrice, double atr5m) {

    double scanWidth = Math.min(
            consolAtrMult * atr5m,          // Default: 0.75 × ATR
            consolPctCap * currentPrice);   // Default: 1.5% × Preis

    double stepSize = scanWidth / 4.0;      // 25% Overlap zwischen Fenstern
    double minDensity = minTouchesPerWindow; // Default: 5 Touches pro Fenster

    // Scan über relevanten Preisbereich (±3 ATR um aktuellen Preis)
    double scanLow = currentPrice - 3.0 * atr5m;
    double scanHigh = currentPrice + 3.0 * atr5m;

    List<DenseRegion> denseRegions = new ArrayList<>();
    DenseRegion current = null;

    for (double price = scanLow; price <= scanHigh; price += stepSize) {
        int touchCount = registry.countValidated(
                price - scanWidth / 2, price + scanWidth / 2, 0.50);

        if (touchCount >= minDensity) {
            if (current == null) {
                current = new DenseRegion(price - scanWidth / 2);
            }
            current.extendTo(price + scanWidth / 2, touchCount);
        } else {
            if (current != null) {
                denseRegions.add(current);
                current = null;
            }
        }
    }

    // Filtere: Nur Regionen mit signifikantem vertikalem Spread
    double minSpread = minSpreadAtrMult * atr5m;  // Default: 0.3 × ATR
    return denseRegions.stream()
            .filter(r -> r.spread() >= minSpread)
            .map(r -> buildConsolidationBand(r, registry))
            .toList();
}
```

### POC-Berechnung (Point of Control)

Der POC ist der touch-gewichtete Schwerpunkt des Bands — das Preislevel mit der
höchsten Touch-Konzentration:

```java
private double computePoc(TouchRegistry registry, double bottom, double top) {
    List<TouchEvent> touches = registry.queryRange(bottom, top);
    double weightedSum = 0.0;
    double totalWeight = 0.0;
    for (TouchEvent touch : touches) {
        double weight = touch.quality();  // Quality als Gewicht
        weightedSum += touch.touchPrice() * weight;
        totalWeight += weight;
    }
    return totalWeight > 0 ? weightedSum / totalWeight : (bottom + top) / 2.0;
}
```

### Integration: Bands ersetzen Cluster-Linien innerhalb der Zone

Wenn ein ConsolidationBand erkannt wird:

1. **Alle Cluster-Level innerhalb des Bands werden entfernt** aus der Zone-Ausgabe
2. **Das Band wird als einzelne Zone exportiert** mit:
   - `center` = POC
   - `band` = (top - bottom) / 2
   - `touchCount` = kumulierte Touches
   - `type` = CONSOLIDATION (neuer Zone-Typ)
3. **Statische Level** (VWAP, OR) die innerhalb des Bands liegen bleiben
   separat erhalten — sie haben eigenständige Bedeutung

### Pipeline-Flow

```
onSnapshot():
    1. updateOpeningRanges()                → OR-Level
    2. detectAndClusterSwingLevels()        → SWING_CLUSTER Level
    3. detectTouches() → TouchRegistry      → Touches räumlich gespeichert
    4. fuseLevelsIntoZones()                → Zones erzeugen
    5. enrichZonesFromRegistry()            → touchCount aus Registry
    6. scoreZones()                         → Confidence berechnen
    7. suppressOverdenseZones()             → NMS (unverändert)
                                               ↓
    8. detectConsolidationBands()           → NEU: Dichte-Scan auf Registry
    9. replaceClusterZonesWithBands()       → NEU: Cluster-Linien → Band-Objekte
                                               ↓
   10. applyOutputPolicy()                 → Threshold + maxZones
```

### Erwartetes Ergebnis für IREN 23.02

**Vorher (4 Cluster-Linien):**
```
41.703  touchCount=5   confidence=0.309  CLUSTER
41.815  touchCount=3   confidence=0.369  CLUSTER
41.929  touchCount=11  confidence=0.320  CLUSTER
42.073  touchCount=41  confidence=0.322  CLUSTER
```

**Nachher (1 ConsolidationBand):**
```
Band: bottom=41.65, top=42.10, poc=42.04
      totalTouches=60, type=CONSOLIDATION
```

4 Linien → 1 Band-Objekt. Klar kommuniziert: "Chop-Zone, handle den Breakout."

### Downstream-Signale

Das ConsolidationBand ändert die Semantik für nachgelagerte Engines:

| Consumer | Signal |
|----------|--------|
| **LLM-Analyst** | "Konsolidierung zwischen $41.65–$42.10, POC bei $42.04. Breakout-Trade bevorzugen." |
| **Rules Engine** | Kein Entry innerhalb des Bands. Entry nur bei Breakout über top oder unter bottom. |
| **Frontend** | Band als gefülltes Rechteck rendern (nicht 4 einzelne Linien) |

### Stabilität (Flickering-Fix)

Das Band-Objekt ist **inherent stabiler** als 3 Repräsentanten-Linien:
- POC ist touch-gewichteter Schwerpunkt → ein neuer Touch verschiebt ihn minimal
- Top/Bottom ändern sich nur wenn Touches AUSSERHALB des bisherigen Bands auftreten
- Kein Sprung von "Lower-Edge ist 41.70" auf "Lower-Edge ist 41.82"
- Das Band wächst monoton (neue Touches können es nur erweitern, nie schrumpfen)

### Konfigurationsparameter (SrProperties)

```java
record ConsolidationProperties(
    double consolAtrMult,         // Scan-Fensterbreite: Default 0.75 × ATR
    double consolPctCap,          // Scan-Fenster-Cap: Default 0.015 (1.5%)
    int minTouchesPerWindow,      // Min Touches für "dicht": Default 5
    double minSpreadAtrMult,      // Min Spread für Band: Default 0.3 × ATR
    double scanRangeAtrMult       // Scan-Bereich: Default 3.0 × ATR um Preis
) {}
```

Namespace: `odin.brain.sr.consolidation.*`

### Risiken und Mitigationen

| Risiko | Mitigation |
|--------|------------|
| Echte separate Level in ein Band gezogen | minSpread + Dichte-Threshold: Band nur bei zusammenhängender hoher Dichte |
| Band wird zu breit (> 2% Preis) | consolPctCap begrenzt Scan-Fenster; Dichte-Lücken unterbrechen das Band |
| Zu wenig Touches für Dichte-Erkennung | Früh in der Session: kein Band erkannt, normale Level-Ausgabe |
| VWAP innerhalb des Bands geht verloren | VWAP bleibt als separate Zone erhalten (statisches Level) |
| Performance: Dichte-Scan pro Snapshot | Scan-Bereich begrenzt (±3 ATR); Bin-basierte Registry ist O(log n) |

### Tests

1. 4 Cluster in $0.37 Spanne mit 60 Touches → ConsolidationBand erkannt
2. 2 Level mit $2 Abstand → kein Band (Dichte-Lücke dazwischen)
3. Band-POC liegt beim touch-gewichteten Schwerpunkt (nicht geometrische Mitte)
4. VWAP innerhalb Band → bleibt als separate Zone erhalten
5. Band wächst monoton: neuer Touch am Rand erweitert, schrumpft nie
6. Frühe Session (< 5 Touches im Scan-Fenster) → kein Band, normale Cluster-Ausgabe
7. Band-Stabilität: 10 aufeinanderfolgende Snapshots → POC bewegt sich um < 0.02
8. Downstream: Band-Zone hat type=CONSOLIDATION, enthält bottom/top/poc
