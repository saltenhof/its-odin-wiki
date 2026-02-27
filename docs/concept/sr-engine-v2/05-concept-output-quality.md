# Mini-Konzept 5: Gesamtqualität der Zone-Ausgabe

**Problem:** 12 Zonen mit niedriger Confidence (max 0.565), zu viele Low-Value-Zonen.
**Quelle:** ChatGPT + Gemini Sparring 2026-02-27

---

## Lösungsansatz: Drei Bausteine

### Baustein 1: Minimum-Confidence-Threshold (Hybrid: statisch + dynamisch)

**Statischer Boden (Noise-Filter):**
```
minConfFloor = 0.30
```
Alles unter 0.30 wird nie ausgegeben (statistisches Rauschen).

**Dynamischer Cutoff (Relative Stärke):**
```
minConf = max(minConfFloor, dynamicFactor × Cmax)
```
wobei `Cmax` = höchste Confidence aller Zonen, `dynamicFactor` = 0.60 (Default).

**Beispiel IREN:** Cmax=0.565 → Threshold = max(0.30, 0.339) = 0.339
→ 5 Noise-Zonen (<0.30) und 2 marginale Zonen (<0.339) werden entfernt.

**Fallback:** Wenn nach Threshold < `minZones` (Default: 4) übrig bleiben,
relaxiere Threshold schrittweise bis minZones erreicht (nie unter Floor).

### Baustein 2: maxZones auf 8 + räumliche Verteilung

**maxZones = 8** statt 12 (reduziert Clutter).

**Spatial Allocation (Gemini-Vorschlag):**
- Teile Zonen in "Above Price" (Resistance) und "Below Price" (Support)
- Reserviere mindestens `minPerSide` (Default: 2) Slots pro Seite
- Fülle restliche Slots nach Confidence

```
supportZones = zones.filter(z -> z.center() < currentPrice)
resistZones  = zones.filter(z -> z.center() >= currentPrice)

// Mindestens 2 pro Seite, Rest nach Confidence
supportSlots = max(minPerSide, maxZones/2 - (resistZones.size() < minPerSide ? minPerSide - resistZones.size() : 0))
resistSlots  = maxZones - supportSlots
```

**Effekt:** An Trendtagen nicht nur Resistance oder nur Support.

### Baustein 3: Confidence-Kalibrierung

**Problem:** Die aktuelle lineare Gewichtung (45% structural + 55% behavioral) drückt
starke Level nach unten. Ein VWAP (Prior 0.85) startet strukturell bei nur 0.38 bevor
behavioral überhaupt zählt.

**Zwei komplementäre Fixes:**

#### Fix 3a: Base+Boost statt gewichteter Durchschnitt (Gemini-Vorschlag)

Statt:
```
confidence = 0.45 × structural + 0.55 × behavioral
```

Besser:
```
confidence = structural + (1.0 - structural) × behavioral × boostWeight
```
`boostWeight` = 0.80 (Default)

**Beispiele:**
| Level | structural | behavioral | Alt (linear) | Neu (Base+Boost) |
|-------|-----------|-----------|-------------|-----------------|
| VWAP, 9 Touches | 0.82 | 0.55 | 0.67 | 0.90 |
| Cluster, 5 Touches | 0.55 | 0.45 | 0.50 | 0.71 |
| Noise, 0 Touches | 0.30 | 0.10 | 0.19 | 0.36 |

**Eigenschaft:** Monoton in beiden Inputs → Ranking bleibt erhalten.
Structural ist jetzt Basis (nie unterschritten), Behavioral boosted nach oben.

#### Fix 3b: Tightness als Blend statt Multiplikator (ChatGPT-Vorschlag)

Statt:
```
behavioral = behavioralCore × tightness
```

Besser:
```
behavioral = behavioralCore × (tightnessFloor + (1 - tightnessFloor) × tightness)
```
`tightnessFloor` = 0.65 (Default)

**Effekt:** Tightness kann behavioral maximal um 35% drücken (statt 100%).
Ranking bleibt stabil (monotone Transformation von tightness).

### Pipeline-Integration

```
scoreZones()                    → Confidence berechnet (mit neuem Base+Boost)
suppressOverdenseZones()        → NMS
compressConsolidationZones()    → Konsolidierung komprimiert (Konzept 2)
                                   ↓
applyOutputPolicy()             → NEU:
    1. minConf-Threshold anwenden (hybrid: Floor + dynamisch)
    2. Spatial Allocation (Support/Resistance-Balance)
    3. maxZones=8 Cap
                                   ↓
Final Output
```

### Konfigurationsparameter

```java
record OutputPolicyProperties(
    double minConfFloor,           // Default: 0.30
    double dynamicThresholdFactor, // Default: 0.60
    int maxZones,                  // Default: 8 (war 12)
    int minZones,                  // Default: 4
    int minPerSide,                // Default: 2
    double boostWeight,            // Default: 0.80
    double tightnessFloor          // Default: 0.65
) {}
```

Namespace: `odin.brain.sr.output.*`

### Erwartetes Ergebnis für IREN 23.02

**Vorher (12 Zonen):**
```
38.96 OR    conf=0.298   → unter Floor 0.30 → ENTFERNT
39.64 OR    conf=0.266   → unter Floor       → ENTFERNT
39.72 OR    conf=0.266   → unter Floor       → ENTFERNT
40.32 OR    conf=0.446   → nach OR-Filter    → PARKED (Problem 1)
40.47 OR    conf=0.358   → nach OR-Filter    → PARKED (Problem 1)
40.93 CL    conf=0.288   → unter Floor       → ENTFERNT
41.07 CL    conf=0.412   → überlebt
41.34 VWAP  conf=0.565   → überlebt (Top)
41.70 CL    conf=0.309   → knapp über dynamisch
41.81 CL    conf=0.369   → nach Konsolidierung → komprimiert (Problem 2)
41.93 CL    conf=0.320   → nach Konsolidierung → komprimiert (Problem 2)
42.07 CL    conf=0.322   → überlebt (41 Touches)
```

**Nachher (geschätzt 5–7 Zonen):**
- 41.34 VWAP (conf ~0.90 nach Rekalibrierung)
- 41.07 CLUSTER (conf ~0.70)
- 42.07 CLUSTER (conf ~0.65, Dominant Konsolidierung)
- 41.70 CLUSTER (conf ~0.55, Lower-Edge Konsolidierung)
- 40.93 CLUSTER (conf ~0.45) — wenn über dynamischem Threshold

Deutlich saubererer, actionable Output.

### Risiken und Mitigationen

| Risiko | Mitigation |
|--------|------------|
| Zu wenig Zonen an strukturlosen Tagen | minZones=4 mit Threshold-Relaxierung |
| Nur Support oder nur Resistance | Spatial Allocation mit minPerSide=2 |
| Confidence-Rekalibrierung ändert Rankings | Base+Boost ist monoton in beiden Inputs |
| Tightness-Floor macht unscharfe Level gut | Floor=0.65 lässt noch 35% Penalty zu |

### Tests

1. Zone mit confidence 0.25 → unter Floor → nicht in Ausgabe
2. Dynamischer Threshold: Cmax=0.80 → Threshold=0.48 → alles unter 0.48 raus
3. minZones: Nur 3 Zonen über Threshold → Threshold relaxiert bis 4 vorhanden
4. Spatial Allocation: 6 Resistance, 2 Support → mindestens 2 Support in Ausgabe
5. Base+Boost Monotonie: structural steigt → confidence steigt
6. Base+Boost Monotonie: behavioral steigt → confidence steigt
7. maxZones=8: nie mehr als 8 Zonen in Ausgabe
8. Tightness-Floor: tightness=0.3 → behavioral wird um ~25% gedrückt (nicht 70%)
