# Mini-Konzept 4: Reconciliation-Drift (Touch-Historie-Verlust)

**Problem:** Cluster-Center-Drift über epsilon → altes Level gelöscht → Live-Touches verloren.
**Quellen:** ChatGPT-Sparring (erweitertes Matching, Grace Period) + Gemini-Review (Ghost-Touch-Kritik, Spatial Touch Registry)

---

## Gemini-Kritik am ursprünglichen ChatGPT-Konzept

Das ursprüngliche Konzept (Merge + Touch-Mitnahme + Center-Glättung) hatte einen
**kritischen blinden Fleck: Ghost Touches.**

> Level driftet von $41.80 auf $41.95, nimmt 5 Touches mit. Das System signalisiert
> jetzt starken Support bei $41.95 — aber die 5 Touches fanden bei $41.80 statt.
> Bei $41.95 gab es in der Historie NIE echte Reaktionen. Ein Trader handelt gegen
> ein "starkes Level" dessen Stärke auf einer falschen Preisachse basiert.

Weitere Kritik:
- **Center-Glättung (α=0.4) ist gefährlich:** S/R ist eine harte Barriere, kein
  gleitender Durchschnitt. Bei Updates alle 5 Bars dauert es ~1 Stunde bis das
  Level nachzieht.
- **Split/Merge-Logik ist over-engineered:** Flip-Flop-Verhalten an Cluster-Rändern,
  schwer debuggbar.
- **Jaccard auf Sliding Windows** ist instabil wenn Pivots aus dem Zeitfenster fallen.

---

## Neuer Lösungsansatz: Spatial Touch Registry + Grace Period

### Kernprinzip

**Touches gehören zum PREIS, nicht zum Level.**

Ein Level "besitzt" keine Touches. Stattdessen gibt es eine globale, räumlich
indizierte Touch-Registry. Jedes Level fragt bei Bedarf ab: "Wie viele validierte
Touches gibt es in meiner Preis-Band-Zone?" Das eliminiert das gesamte
Migrationsproblem — und damit Ghost Touches — an der Wurzel.

### Architektur: TouchRegistry (neu)

```java
/**
 * Spatial index of all touch events, keyed by price bin.
 * Levels query the registry for touches within their band
 * instead of owning touch events directly.
 */
public class TouchRegistry {

    private static final double BIN_SIZE = 0.01;  // $0.01 Bins

    private final NavigableMap<Integer, List<TouchEvent>> bins = new TreeMap<>();

    /** Record a touch at a specific price. */
    public void record(TouchEvent touch) {
        int bin = priceToBin(touch.touchPrice());
        bins.computeIfAbsent(bin, k -> new ArrayList<>()).add(touch);
    }

    /** Query all touches within a price range. */
    public List<TouchEvent> queryRange(double lower, double upper) {
        int loBin = priceToBin(lower);
        int hiBin = priceToBin(upper);
        List<TouchEvent> result = new ArrayList<>();
        for (Map.Entry<Integer, List<TouchEvent>> entry :
                bins.subMap(loBin, true, hiBin, true).entrySet()) {
            result.addAll(entry.getValue());
        }
        return result;
    }

    /** Count validated touches (quality >= threshold) in range. */
    public int countValidated(double lower, double upper, double minQuality) {
        return (int) queryRange(lower, upper).stream()
                .filter(t -> t.quality() >= minQuality)
                .count();
    }

    private int priceToBin(double price) {
        return (int) Math.round(price / BIN_SIZE);
    }
}
```

### Wie Levels die Registry nutzen

Statt `level.touchCount()` → `registry.countValidated(level.lower(), level.upper(), 0.50)`.

```java
// Vorher (Touches als Level-Property):
int touches = level.validatedTouchCount();

// Nachher (Touches aus Registry):
double lower = level.center() - level.band();
double upper = level.center() + level.band();
int touches = touchRegistry.countValidated(lower, upper, VALIDATED_TOUCH_QUALITY_MIN);
```

**Effekt bei Drift:** Level driftet von $41.80 auf $41.95 → es fragt jetzt den
Bereich $41.90–$42.00 ab → sieht nur die Touches die dort tatsächlich stattfanden.
Die 5 Touches bei $41.80 bleiben in der Registry erhalten — wenn das Level
zurückdriftet, sieht es sie sofort wieder.

### Touch-Erzeugung (wer schreibt in die Registry?)

Zwei Quellen — beide schreiben in die globale TouchRegistry statt in ein Level:

**1. Live Touch-Erkennung (`detectTouches`):**
```java
// Aktuell: level.addTouch(touchEvent)
// Neu:     touchRegistry.record(touchEvent)
```
Die Touch-Erkennung prüft weiterhin Proximity zu aktiven Levels.
Aber der Touch wird räumlich gespeichert, nicht am Level.

**2. Retroaktives Touch-Seeding (`seedTouchesFromPivots`):**
```java
// Aktuell: newLevel.addTouch(touchEvent)
// Neu:     touchRegistry.record(touchEvent)
```
Identisch — Seeded Touches landen in der gleichen Registry.

### Was bleibt von `reconcileSwingClusters()`

Reconciliation wird **massiv vereinfacht**. Es geht nur noch um Level-Identität
(wann ist ein Cluster "derselbe"), nicht mehr um Touch-Migration:

**1. Erweitertes Matching (beibehalten, vereinfacht):**
```
matchable ⟺ abs(newCenter - existingCenter) <= 2 × epsilon
             ODER interval-overlap > 0
```
Kein Jaccard nötig (war instabil bei Sliding Windows laut Gemini).

**2. Bei Match: Center + Band updaten**
```java
existing.updateCenter(newCluster.center())  // hart, KEINE Glättung
existing.updateBand(newCluster.band())
existing.updateSourcePivots(newCluster.pivots())
// Touch-Registry ist unverändert — keine Migration nötig!
```
Center springt auf den neuen DBSCAN-Center (Gemini: S/R ist eine harte Barriere).

**3. Grace Period für Unmatched Level (beibehalten):**
```
existing.state = ORPHANED
existing.orphanCycles++
```
- Bleibt in der aktiven Level-Liste (Touch-Erkennung läuft weiter)
- Löschung erst wenn: `orphanCycles >= 3 AND preisfern`
- Bei Re-Match: `state = ACTIVE, orphanCycles = 0`

**4. Split/Merge-Logik: ENTFERNT** (over-engineered, Flip-Flop-Risiko).
- Split: zweiter Cluster wird einfach neues Level (Standard-Verhalten)
- Merge: schwächeres Level wird durch Grace Period natürlich ausgetrocknet

### Auswirkungen auf andere Komponenten

| Komponente | Änderung |
|------------|----------|
| `SrLevel` | `touches`-Liste ENTFERNT; `addTouch()` ENTFERNT; `validatedTouchCount()` delegiert an Registry |
| `SrZone` | `totalValidatedTouchCount()` aggregiert über Registry statt über Level |
| `SupportLevelConfidence` | behavioral.touchCount fragt Registry ab |
| `SrLevelEngine` | Hält `TouchRegistry` als Feld; `detectTouches()` und `seedTouchesFromPivots()` schreiben in Registry |
| `TouchEvent` | Unverändert (immutables Record) |
| `SrSnapshot` | Unverändert (Zones enthalten berechneten touchCount aus Registry) |

### Implementierung im Pipeline-Flow

```
onSnapshot():
    1. updateOpeningRanges()           → OR-Level erzeugen (unverändert)
    2. detectAndClusterSwingLevels()   → DBSCAN + Reconciliation (vereinfacht, ohne Touch-Migration)
    3. detectTouches()                 → schreibt in TouchRegistry (statt in Level)
    4. fuseLevelsIntoZones()           → Zones erzeugen
    5. enrichZonesFromRegistry()       → NEU: touchCount pro Zone aus Registry abfragen
    6. scoreZones()                    → Confidence berechnen (mit Registry-basierten Touches)
    7. suppressOverdenseZones()        → NMS
```

### Konfigurationsparameter

```java
record ReconciliationProperties(
    double matchDistanceMultiplier,    // Default: 2.0 (× epsilon)
    int maxOrphanCycles,               // Default: 3
    double farGateAtrMult              // Default: 3.0
) {}
```

Namespace: `odin.brain.sr.reconciliation.*`

**Entfernt** gegenüber ChatGPT-Konzept:
- `centerSmoothingAlpha` (Gemini: S/R ist hart, keine Glättung)
- `gapMaxMultiplier` (Jaccard entfernt, nur Distance + Overlap)
- `noRecentTouchBars` (überflüssig — Touches sind in Registry, Level-Löschung über orphanCycles + Distanz)

### Erwartetes Verhalten für $41.80 → $41.95 Drift

**Vorher (aktuelles System):**
1. Level bei $41.80, 5 Live-Touches direkt am Level
2. DBSCAN erzeugt neuen Cluster bei $41.95 (Drift > epsilon)
3. Level $41.80 gelöscht → 5 Touches verloren
4. Neues Level bei $41.95 → 0 Touches

**Nachher (Spatial Touch Registry):**
1. Level bei $41.80. 5 Live-Touches in der Registry bei $41.78–$41.82
2. DBSCAN erzeugt neuen Cluster bei $41.95 (Drift > epsilon, < 2×epsilon)
3. Match gefunden → Level-Center auf $41.95 aktualisiert, Band ±$0.05
4. Level fragt Registry: "Touches in $41.90–$42.00?" → 0 oder wenige
5. Die 5 Touches bei $41.80 bleiben in der Registry
6. Wenn später ein neues Level bei $41.80 entsteht, sieht es sofort die 5 Touches

**Kein Ghost-Touch-Problem:** Das Level bei $41.95 behauptet NICHT mehr, 5 Touches zu
haben. Es hat genau so viele Touches wie tatsächlich bei $41.90–$42.00 stattfanden.

### Vergleich der Ansätze

| Aspekt | ChatGPT (Merge+Transfer) | Gemini (Spatial Registry) |
|--------|--------------------------|---------------------------|
| Ghost Touches | Ja — falsches Signal bei Drift | Nein — Touches sind ortsgebunden |
| Komplexität | Hoch (Merge, Transfer, Dedup, Glättung, Split/Merge) | Niedrig (Registry + Query) |
| Reconciliation | Komplex (Touch-Migration) | Einfach (nur Center/Band update) |
| Datenintegrität | Touches driften mit Level | Touches bleiben wo sie stattfanden |
| Debug-Fähigkeit | Schwer (woher kam dieser Touch?) | Einfach (Registry ist Single Source of Truth) |
| Performance | O(touches) pro Merge | O(log n) pro Range-Query (TreeMap) |

### Risiken und Mitigationen

| Risiko | Mitigation |
|--------|------------|
| Registry wächst über den Tag | Nur RTH-Touches; typisch < 1000 pro Session |
| Bin-Größe zu grob/zu fein | $0.01 ist Tick-Size für US-Equities; konfigurierbar |
| Erweitertes Matching verbindet separate Level | 2×epsilon als Obergrenze; Side-Constraint (Lows nur mit Lows) |
| Orphaned Level akkumulieren | maxOrphanCycles=3 + farGate → Cleanup |
| Neues Level bei $41.95 hat 0 Touches (sieht "schwach" aus) | Korrekt! Es IST schwach dort — das ist die Wahrheit, nicht ein Bug |

### Tests

1. Touch bei $41.80 → Registry-Query $41.75–$41.85 → enthält den Touch
2. Touch bei $41.80 → Registry-Query $41.90–$42.00 → enthält den Touch NICHT
3. Level driftet $41.80 → $41.95 → touchCount ändert sich (Registry-basiert, kein Transfer)
4. Level driftet $41.95 → $41.80 zurück → sieht die Original-Touches wieder
5. Zwei Level mit überlappenden Bands → teilen sich Touches korrekt
6. Orphaned Level: kein Match für 3 Zyklen + preisfern → gelöscht
7. Orphaned Level: kein Match für 2 Zyklen, dann Re-Match → ACTIVE, kein Datenverlust
8. Registry nach voller Session: alle Touches vorhanden, keine Duplikate
9. Seeded Touch + Live Touch am selben Preis → beide in Registry, kein Konflikt
