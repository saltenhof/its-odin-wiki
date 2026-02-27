# Mini-Konzept 1: OR-Level-Überflutung

**Problem:** 5 von 12 Zonen sind OR-Level, davon 3 mit 0 Touches 6–8% unter der Handelszone.
**Quellen:** ChatGPT-Sparring (Relevanz-Gate, Quota) + Gemini-Review (Registry-Konsistenz, Hysterese, MID-Semantik, Quota-Kollision, NMS-Schutz)

---

## Gemini-Kritik am ursprünglichen ChatGPT-Konzept

Das ursprüngliche Konzept (PROTECTED/ACTIVE/PARKED mit statischem touchCount + eigenem Quota)
hatte fünf Probleme:

**1. touchCount als statisches Flag widerspricht Konzept 4:**
> In der Spatial Touch Registry besitzen Level keine Touches mehr. Sie fragen die
> Registry dynamisch ab. Wenn du OR-Level beim Erzeugen mit touchCount=0 taggst —
> wann und wie werden sie auf PROTECTED geupdatet? PROTECTED/ACTIVE muss aus der
> TouchRegistry berechnet werden, nicht als Flag gesetzt.

**2. Kein Hysterese-Mechanismus → Flickering:**
> Preis schwankt um genau 4.0 ATR Abstand. Tick 1: 3.99 ATR (ACTIVE). Tick 2:
> 4.01 ATR (PARKED). Das Level taucht in der UI und in der Downstream-Logik im
> Sekundentakt auf und verschwindet wieder. Dasselbe Flickering-Problem wie bei
> Konzept 2 (Konsolidierungszonen).

**3. MID-Level gegen ATR_5m ist mathematischer Unfug:**
> Die Opening Range (z.B. 3 Bars à 5 Min) hat eine Range vom absoluten High zum
> absoluten Low dieser Phase. Die ATR_5m ist die Volatilität einer einzelnen
> 5m-Kerze. Die Range von 3 Kerzen ist fast immer größer als 1.5× eine Einzelkerze.
> Effekt: MID wird fast nie generiert. Referenz muss Daily-ATR oder historische
> OR-Range sein.

**4. OR-Quota kollidiert mit Konzept 5 (Output Quality):**
> Konzept 5 etabliert ein sauberes, globales System: Spatial Allocation + Confidence
> Thresholds. Ein lokales OR-Quota dazwischengeklemmt erzeugt Kollisionen. Was
> passiert, wenn das OR-Quota 2 Resistance-Level durchwinkt, aber Konzept 5 dringend
> Support-Slots füllen muss? Nicht nötig: ACTIVE OR-Level ohne Touches bekommen nur
> Base Confidence und werden von Konzept 5 natürlich runtergerankt.

**5. keepAlways blockiert NMS-Merging:**
> OR-High und VWAP auf fast dem gleichen Preis, beide PROTECTED → NMS kann keines
> unterdrücken → zwei Zonen im Abstand von 2 Cent. keepAlways darf NMS-Merge
> niemals verhindern. Räumlich identische Level müssen immer gemerged werden.

---

## Neuer Lösungsansatz: Dynamisches Relevanz-Gate mit Hysterese

### Kernprinzip

**OR-Level-Relevanz wird dynamisch berechnet, nicht statisch getaggt.**

Bei jeder Bar-Aktualisierung fragt das OR-Level die TouchRegistry ab und berechnet
seinen Status. Kein Sonder-Quota — die globale Output Policy (Konzept 5) reguliert
OR-Level natürlich über ihre Confidence.

### Status-Bestimmung (dynamisch, pro Bar)

```java
/**
 * Computes the relevance status of an OR level dynamically.
 * Uses the TouchRegistry for touch count (Konzept 4 consistency)
 * and hysteresis thresholds for stability (no flickering).
 */
public enum OrRelevanceStatus {
    PROTECTED,   // Touches > 0 → Level hat Markt-Bestätigung
    ACTIVE,      // 0 Touches, aber preisnah → Kandidat
    PARKED       // 0 Touches und preisfern → aus Pipeline entfernt
}
```

| Status    | Bedingung                                              | Pipeline-Verhalten |
|-----------|--------------------------------------------------------|--------------------|
| PROTECTED | `registry.countValidated(lower, upper, minQuality) > 0` | Normaler Kandidat in NMS + Scoring |
| ACTIVE    | 0 validierte Touches UND preisnah (Hysterese)          | Normaler Kandidat (niedrige Confidence durch Konzept 5) |
| PARKED    | 0 validierte Touches UND preisfern (Hysterese)         | Komplett aus Pipeline entfernt |

**Schlüssel:** PROTECTED und ACTIVE sind keine Sonderfälle — sie gehen als ganz normale
Kandidaten durch NMS und Konzept 5. Kein keepAlways, kein Sonder-Quota.

### Hysterese-Gate (Flickering-Schutz)

Statt eines einzelnen Schwellwerts verwendet das Gate asymmetrische Schwellen:

```java
/**
 * Hysteresis gate for OR level proximity. Uses asymmetric thresholds
 * to prevent flickering when price oscillates near the boundary.
 *
 * ACTIVE → PARKED: only when distance exceeds parkThreshold (wider)
 * PARKED → ACTIVE: only when distance drops below activateThreshold (narrower)
 *
 * Gap between thresholds = dead zone where no state change occurs.
 */
private boolean isNearPrice(double levelPrice, double currentPrice,
                            double atr5m, OrRelevanceStatus currentStatus) {

    double distance = Math.abs(levelPrice - currentPrice);
    double activateThreshold = Math.min(
            activateAtrMult * atr5m,        // Default: 4.0 × ATR
            activatePctCap * currentPrice);  // Default: 3.0%
    double parkThreshold = Math.min(
            parkAtrMult * atr5m,             // Default: 5.0 × ATR
            parkPctCap * currentPrice);      // Default: 4.0%

    if (currentStatus == OrRelevanceStatus.PARKED) {
        // Muss unter den engeren Threshold fallen um ACTIVE zu werden
        return distance <= activateThreshold;
    } else {
        // Muss über den weiteren Threshold steigen um PARKED zu werden
        return distance <= parkThreshold;
    }
}
```

**Beispiel:** ATR_5m = $0.25, Preis = $42.00
- `activateThreshold = min(4.0 × 0.25, 0.03 × 42.00) = min(1.00, 1.26) = $1.00`
- `parkThreshold = min(5.0 × 0.25, 0.04 × 42.00) = min(1.25, 1.68) = $1.25`
- OR-Level bei $40.90 (Abstand $1.10):
  - Wenn vorher ACTIVE: bleibt ACTIVE ($1.10 < $1.25 parkThreshold)
  - Wenn vorher PARKED: bleibt PARKED ($1.10 > $1.00 activateThreshold)
- Dead Zone: $1.00–$1.25 — kein Zustandswechsel, egal in welcher Richtung

### MID-Level: Konditionale Generierung (korrigiert)

OR_MID wird **nur erzeugt** wenn die Opening Range relativ zum Tagesverhalten "eng" ist:

```
OR_range = OR_HIGH - OR_LOW
generateMid ⟺ OR_range <= midRangeMaxDailyAtrFraction × Daily_ATR_estimate
```

**Parameter:** `midRangeMaxDailyAtrFraction`: 0.30 (Default)

**Daily-ATR-Schätzung:** In den ersten RTH-Minuten steht keine vollständige Daily ATR
zur Verfügung. Proxy:

```java
/**
 * Estimates the expected daily range from early session data.
 * Uses the larger of: ATR_5m scaled to full session, or observed range so far.
 */
double dailyAtrEstimate = Math.max(
    atr5m * Math.sqrt(78.0),    // 78 × 5m Bars in RTH, √ für Volatilitätsskalierung
    sessionHighSoFar - sessionLowSoFar  // Beobachtete Range
);
```

**Begründung (Gemini-Korrektur):** Die OR-Range von 3 Bars à 5 Min ist fast immer
größer als 1.5× ATR_5m (einer einzelnen Kerze). Die Referenz muss die erwartete
Tagesrange sein, nicht die Einzelkerzen-Volatilität.

**Beispiel:** Daily ATR estimate = $1.50, `midRangeMaxDailyAtrFraction = 0.30`
→ MID nur wenn OR_range ≤ $0.45. Bei einer OR von $0.60 → kein MID (Range zu breit).

### NMS-Verhalten: Kein Sonderschutz

```java
// PROTECTED OR-Level sind KEINE keepAlways-Kandidaten.
// Sie gehen als normale Kandidaten durch NMS.
// Räumlich identische Level werden IMMER gemerged,
// egal ob OR, VWAP oder Cluster.

// Vorher (ChatGPT): keepAlways.add(protectedOrZones);
// Nachher: candidates.add(protectedOrZones);  // ganz normal
```

**Einzige Ausnahme:** VWAP behält keepAlways-Status (hat eigenständige, dynamische
Bedeutung die nicht in ein Cluster-Level gemerged werden sollte).

### Keine Sonder-Quota (Gemini: "Wirf das OR-Quota komplett raus")

Die Output Policy (Konzept 5) reguliert OR-Level natürlich:

1. **PARKED OR-Level:** Komplett aus Pipeline entfernt (vor Scoring)
2. **ACTIVE OR-Level (0 Touches):** Gehen normal in die Pipeline. Bekommen nur
   Base Confidence (niedrig). Werden durch den dynamischen Threshold aus Konzept 5
   (`max(minConfFloor, dynamicFactor × Cmax)`) natürlich aussortiert wenn es
   stärkere Level gibt.
3. **PROTECTED OR-Level (Touches > 0):** Bekommen durch TouchRegistry-Touches
   einen Behavioral Boost. Konkurrieren auf Augenhöhe mit Cluster-Leveln.

**Effekt:** Kein `unconfirmedOrMax`-Parameter nötig. Keine Quota-Kollision mit
Spatial Allocation.

### Pipeline-Flow

```
updateOpeningRanges()             → OR-Level werden IMMER erzeugt (keine Änderung)
addOrUpdateStaticLevels()         → OR-Level werden IMMER als SrLevel angelegt
                                     ↓
computeOrRelevanceStatus()        → NEU: Registry-Query + Hysterese pro OR-Level
                                     PARKED → aus Pipeline entfernt
                                     PROTECTED/ACTIVE → weiter als normale Kandidaten
                                     ↓
fuseLevelsIntoZones()             → Alle verbleibenden Level (OR, Cluster, VWAP)
enrichZonesFromRegistry()         → touchCount aus TouchRegistry (Konzept 4)
scoreZones()                      → Confidence mit Base+Boost (Konzept 5)
suppressOverdenseZones()          → NMS: KEIN keepAlways für OR, normales Merging
                                     ↓
applyOutputPolicy()               → Konzept 5: Threshold + Spatial Allocation + maxZones
                                     OR-Level konkurrieren gleichberechtigt
```

### Konfigurationsparameter (SrProperties)

```java
record OrRelevanceProperties(
    double activateAtrMult,              // Hysterese: PARKED→ACTIVE: Default 4.0
    double activatePctCap,               // Hysterese: PARKED→ACTIVE: Default 0.03 (3%)
    double parkAtrMult,                  // Hysterese: ACTIVE→PARKED: Default 5.0
    double parkPctCap,                   // Hysterese: ACTIVE→PARKED: Default 0.04 (4%)
    double midRangeMaxDailyAtrFraction   // MID nur wenn OR_range <= X × Daily ATR: Default 0.30
) {}
```

Namespace: `odin.brain.sr.or-relevance.*`

**Entfernt** gegenüber ChatGPT-Konzept:
- `unconfirmedOrMax` (Quota entfernt — Konzept 5 reguliert natürlich)
- Einfacher `atrMultiplier` (ersetzt durch asymmetrische Hysterese-Schwellen)
- `midRangeMaxAtr` (ersetzt durch Daily-ATR-basierte Referenz)

### Erwartetes Ergebnis für IREN 23.02

**Kontext:** Preis um $42.00, ATR_5m ≈ $0.25
- `activateThreshold = min(1.00, 1.26) = $1.00` → OR-Level > $1.00 entfernt = unter $41.00
- `parkThreshold = min(1.25, 1.68) = $1.25` → Dead Zone $41.00–$40.75

| Zone       | Abstand | Status    | Aktion |
|------------|---------|-----------|--------|
| $38.96 OR  | $3.04   | PARKED    | Entfernt (weit unter Threshold) |
| $39.64 OR  | $2.36   | PARKED    | Entfernt |
| $39.72 OR  | $2.28   | PARKED    | Entfernt |
| $40.32 OR  | $1.68   | PARKED    | Entfernt |
| $40.47 OR  | $1.53   | PARKED    | Entfernt |

**Alle 5 OR-Level werden als PARKED klassifiziert.** Das gibt 5 freie Slots für
Cluster-Level mit echten Touches.

Wenn der Preis später auf $41.00 fällt: OR bei $40.47 hat Abstand $0.53 → unter
`activateThreshold` → wird ACTIVE. Wenn es dann Touches bekommt → PROTECTED.

### Interaktion mit anderen Konzepten

| Konzept | Interaktion |
|---------|-------------|
| Konzept 2 (Konsolidierung) | OR-Level innerhalb eines ConsolidationBands werden normal gemerged |
| Konzept 3 (Dynamic Scoring) | OR-Touch-Quality wird identisch dynamisch berechnet |
| Konzept 4 (Spatial Registry) | OR-Level fragt Registry ab für PROTECTED/ACTIVE-Bestimmung |
| Konzept 5 (Output Quality) | ACTIVE OR-Level bekommen niedrige Base Confidence → natürliche Regulierung |
| Konzept 6 (Band-API) | OR-Level sind immer `zoneType=LEVEL` (nie BAND) |

### Risiken und Mitigationen

| Risiko | Mitigation |
|--------|------------|
| Preis kehrt spät zum OR zurück | PARKED→ACTIVE Transition dynamisch über Hysterese |
| Hysterese-Lücke zu breit | parkAtrMult - activateAtrMult = 1.0 ATR → moderater Dead Zone |
| ATR instabil in ersten Minuten | min(ATR, %) nimmt konservativeren Wert; Daily-ATR-Proxy |
| PROTECTED OR und VWAP am gleichen Preis | NMS merged sie normal (kein keepAlways für OR) |
| Alle OR-Level PARKED → keine OR-Info | Korrekt: wenn alle OR weit vom Preis entfernt, sind sie irrelevant |
| MID fast nie generiert | Daily-ATR-Referenz statt 5m-ATR → realistischer Threshold |

### Tests

1. OR-Level 5% vom Preis entfernt + 0 Touches → PARKED, nicht in Pipeline
2. OR-Level 1% vom Preis entfernt + 0 Touches → ACTIVE, niedrige Confidence
3. OR-Level 5% entfernt + 2 Touches (Registry) → PROTECTED, konkurriert normal
4. Hysterese: ACTIVE bei Abstand 4.5 ATR → bleibt ACTIVE (< parkThreshold)
5. Hysterese: PARKED bei Abstand 4.5 ATR → bleibt PARKED (> activateThreshold)
6. PROTECTED OR + VWAP ±2 Cent → NMS merged normal (kein keepAlways-Schutz)
7. Enge OR (< 0.30 × Daily ATR) → MID wird generiert
8. Breite OR (> 0.30 × Daily ATR) → MID wird NICHT generiert
9. Preis nähert sich geparktem OR → PARKED→ACTIVE Transition bei activateThreshold
10. ACTIVE OR ohne Touches → unter dynamischem Confidence-Threshold aus Konzept 5 → nicht in Ausgabe
