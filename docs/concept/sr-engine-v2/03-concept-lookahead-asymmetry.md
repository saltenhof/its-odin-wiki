# Mini-Konzept 3: Lookahead-Asymmetrie bei Touch-Seeding

**Problem:** Geseedete Touches werden mit Zukunftsdaten bewertet, Live-Touches nicht → systematische Quality-Verzerrung.
**Quellen:** ChatGPT-Sparring (Maturing Touch Quality) + Gemini-Review (Lag-Kritik, Immutable Events, Dynamic Scoring)

---

## Gemini-Kritik am ursprünglichen ChatGPT-Konzept

Das "Maturing Touch Quality"-Modell (pending Touches die über L Bars nachreifen) hatte
drei fundamentale Probleme:

**1. Pending Touches sind ein Trading-Anti-Pattern:**
> L=3 auf 1m-Bars = 3 Minuten Lag (auf 5m-Bars: 15 Minuten). Ein starker Bounce
> will JETZT gehandelt werden. Das System bestraft die Live-Engine für etwas das sie
> naturgemäß nicht wissen kann. Im Intraday-Trading ist ein 15-Minuten-Lag tödlich.

**2. Mutabilität zerstört die Registry:**
> Pending Touches machen Registry-Einträge veränderlich. Pro Bar müsste die Engine
> durch die Registry iterieren, alle Pending Touches suchen, historische Bars
> abfragen, Quality neu berechnen und das Objekt updaten. Das widerspricht dem
> Spatial-Registry-Design (immutable Events) und ist ein Performance-Albtraum.

**3. Touch ≠ Follow-Through:**
> Die "Quality"-Metrik presst zwei unterschiedliche Marktkonzepte zusammen:
> - Das Touch-Event (T=0): Der Moment wo der Preis das Level trifft
> - Die Reaktion (T>0): Wie stark der Markt danach abweist
> Diese gehören getrennt, nicht in eine einzige mutable Zahl.

---

## Neuer Lösungsansatz: Immutable Events + Dynamic Scoring at Query Time

### Kernprinzip

**Touches sind immutable rohe Events. Quality wird zum Abfragezeitpunkt berechnet,
nicht beim Schreiben.**

Ein Touch-Event in der Spatial Registry enthält nur Fakten die bei T=0 bekannt sind.
Wenn die Engine ein Level bewertet, berechnet sie die Stärke dynamisch — historische
Touches haben natürlich Follow-Through (weil er schon passiert ist), Live-Touches
werden mit sofort verfügbaren T=0-Metriken bewertet. Keine Asymmetrie, kein Lag,
keine Pending-Logik.

### TouchEvent: Immutable, nur T=0-Fakten

```java
/**
 * Immutable record of a price-level touch event.
 * Contains only information available at the moment of the touch (T=0).
 * Quality/follow-through is computed dynamically at query time, not stored.
 */
public record TouchEvent(
    Instant timestamp,        // Zeitpunkt des Touch
    int barIndex,             // 1m-Bar-Index im Session-Kontext
    double touchPrice,        // Preis am Touch-Punkt (low für Support, high für Resistance)
    double barHigh,           // OHLC des Touch-Bars
    double barLow,
    double barClose,
    long barVolume,
    double priceVelocity,     // Preisänderung in den letzten N Bars VOR dem Touch
    double volumeRatio        // Volumen des Touch-Bars / Durchschnittsvolumen
) {}
```

**Entfernt** gegenüber der alten Version:
- `quality` (wird dynamisch berechnet)
- `effectiveLookaheadBars` (gibt es nicht mehr)
- `isPending` (gibt es nicht mehr)

**Neu:**
- `priceVelocity` — wie schnell sich der Preis zum Level bewegt hat (T=0-Metrik)
- `volumeRatio` — Volumen-Spike am Touch-Bar relativ zum Durchschnitt (T=0-Metrik)

### Dynamic Quality Scoring: Berechnung zum Abfragezeitpunkt

Wenn die Engine ein Level bewertet, fragt sie die Registry ab und berechnet Quality
für jeden Touch dynamisch:

```java
/**
 * Computes touch quality dynamically based on information available
 * up to the current bar index. Eliminates lookahead asymmetry because
 * the same function is used for both historical and recent touches.
 *
 * For historical touches: follow-through is naturally available (it already happened).
 * For recent touches: only T=0 metrics are available (no lag, no penalty).
 */
public static double computeDynamic(
        TouchEvent touch,
        List<Bar> oneMinBars,
        int currentBarIndex,
        double levelCenter,
        double atr1m) {

    int age = currentBarIndex - touch.barIndex();

    // === T=0 Komponenten (immer verfügbar) ===
    double proximity = computeProximity(touch.touchPrice(), levelCenter, atr1m);
    double volumeSignal = computeVolumeSignal(touch.volumeRatio());
    double velocitySignal = computeVelocitySignal(touch.priceVelocity(), atr1m);

    double t0Score = T0_PROXIMITY_WEIGHT * proximity       // 0.50
                   + T0_VOLUME_WEIGHT * volumeSignal        // 0.30
                   + T0_VELOCITY_WEIGHT * velocitySignal;   // 0.20

    // === Follow-Through Komponente (wächst natürlich mit dem Alter) ===
    double followThrough = 0.0;
    if (age >= 1) {
        // Bars nach dem Touch sind verfügbar — KEINE Asymmetrie, weil wir
        // nur bis currentBarIndex schauen, nie darüber hinaus
        int lookEnd = Math.min(touch.barIndex() + FOLLOW_THROUGH_HORIZON, currentBarIndex);
        List<Bar> afterTouch = oneMinBars.subList(touch.barIndex() + 1, lookEnd + 1);
        followThrough = computeFollowThrough(afterTouch, touch, levelCenter, atr1m);
    }

    // === Kombination: T=0 dominiert bei frischen Touches, Follow-Through bei alten ===
    double maturityBlend = Math.min(1.0, (double) age / FOLLOW_THROUGH_HORIZON);
    // maturityBlend: 0.0 bei age=0, 1.0 bei age >= HORIZON

    double quality = (1.0 - maturityBlend * FOLLOW_THROUGH_MAX_WEIGHT) * t0Score
                   + maturityBlend * FOLLOW_THROUGH_MAX_WEIGHT * followThrough;

    return SrMath.clamp01(quality);
}
```

### Wie die Asymmetrie verschwindet

| Situation | age | T=0 Score | Follow-Through | Gesamtqualität |
|-----------|-----|-----------|----------------|----------------|
| Live-Touch (gerade erkannt) | 0 | 100% T=0-Metriken | 0% (nicht verfügbar) | Sofort actionable |
| Touch vor 1 Bar | 1 | ~85% T=0 | ~15% Follow-Through | Leicht verbessert |
| Touch vor 3 Bars (HORIZON) | 3 | ~60% T=0 | ~40% Follow-Through | Voll ausgereift |
| Seeded Touch (30 Bars alt) | 30 | ~60% T=0 | ~40% Follow-Through | Identisch mit 3-Bar |

**Schlüssel:** Ein Seeded Touch und ein Live-Touch am selben Bar-Index, abgefragt zum
selben Zeitpunkt, liefern **exakt die gleiche Quality**. Die Funktion ist deterministisch
und hängt nur von (touchBarIndex, currentBarIndex, Bars) ab.

### Kein Pending-State, keine Mutabilität

- Registry-Einträge sind **immutable** — einmal geschrieben, nie verändert
- Quality wird **on-the-fly** berechnet bei jeder Abfrage
- Kein `updatePendingTouches()`-Step in der Pipeline
- Kein `isPending`-Flag, kein `effectiveLookaheadBars`

### T=0-Metriken im Detail

**Proximity** (wie nah war der Touch am Level):
```
proximity = 1.0 - clamp01(abs(touchPrice - levelCenter) / (2 × atr1m))
```
Perfekter Touch am Center = 1.0, weiter weg sinkt linear.

**Volume Signal** (war der Touch-Bar volumenstark):
```
volumeSignal = satExp(volumeRatio - 1.0, 2.0)
```
volumeRatio = 1.0 (durchschnittlich) → Signal ≈ 0. Doppeltes Volumen → Signal ≈ 0.39.
Hoher Spike → Signal → 1.0. Signalisiert institutionelle Aktivität am Level.

**Velocity Signal** (wie schnell näherte sich der Preis):
```
velocitySignal = clamp01(abs(priceVelocity) / (2 × atr1m))
```
Schnelle Annäherung → höheres Signal (der Preis wurde "zum Level gezogen").

### Follow-Through (nur wenn verfügbar)

```java
private static double computeFollowThrough(
        List<Bar> afterBars, TouchEvent touch, double levelCenter, double atr1m) {
    if (afterBars.isEmpty()) return 0.0;

    // Maximale Entfernung vom Level in den Bars nach dem Touch
    double maxSeparation = 0.0;
    for (Bar bar : afterBars) {
        double dist = Math.abs(bar.close() - levelCenter);
        maxSeparation = Math.max(maxSeparation, dist);
    }
    return SrMath.clamp01(maxSeparation / (3.0 * atr1m));
}
```

Misst wie weit der Preis nach dem Touch vom Level wegging (Bounce-Stärke).

### Auswirkungen auf Seeding

`seedTouchesFromPivots()` wird **massiv vereinfacht**:

```java
// Vorher: Quality berechnen mit Lookahead, TouchEvent mit quality erstellen
// Nachher: Nur rohes Event in Registry schreiben

TouchEvent touch = new TouchEvent(
    pivotBar.closeTime(),
    pivotBarIndex,
    touchPrice,
    pivotBar.high(), pivotBar.low(), pivotBar.close(), pivotBar.volume(),
    computeVelocity(oneMinBars, pivotBarIndex),
    computeVolumeRatio(pivotBar, avgVolume));

touchRegistry.record(touch);
// Fertig. Keine Quality-Berechnung hier.
```

### Auswirkungen auf Level-Scoring

Wenn ein Level bewertet wird, fragt es die Registry ab und berechnet Quality
dynamisch für jeden Touch:

```java
List<TouchEvent> touches = touchRegistry.queryRange(level.lower(), level.upper());

double totalQuality = 0.0;
int validatedCount = 0;
for (TouchEvent touch : touches) {
    double quality = TouchQuality.computeDynamic(
            touch, oneMinBars, currentBarIndex, level.center(), atr1m);
    totalQuality += quality;
    if (quality >= VALIDATED_TOUCH_QUALITY_MIN) {
        validatedCount++;
    }
}
```

### Konstanten

```java
static final int FOLLOW_THROUGH_HORIZON = 3;       // Bars bis volle Reife
static final double FOLLOW_THROUGH_MAX_WEIGHT = 0.40; // Max 40% Follow-Through
static final double T0_PROXIMITY_WEIGHT = 0.50;
static final double T0_VOLUME_WEIGHT = 0.30;
static final double T0_VELOCITY_WEIGHT = 0.20;
```

### Konfigurationsparameter (SrProperties)

```java
record TouchScoringProperties(
    int followThroughHorizonBars,     // Default: 3
    double followThroughMaxWeight,    // Default: 0.40
    double t0ProximityWeight,         // Default: 0.50
    double t0VolumeWeight,            // Default: 0.30
    double t0VelocityWeight           // Default: 0.20
) {}
```

Namespace: `odin.brain.sr.touch-scoring.*`

### Vergleich der Ansätze

| Aspekt | ChatGPT (Maturing/Pending) | Gemini-inspiriert (Dynamic Scoring) |
|--------|---------------------------|-------------------------------------|
| Registry-Mutabilität | Mutable (pending → reif) | Immutable (Events ändern sich nie) |
| Lag bei Live-Touches | L Bars (3–15 Min) | 0 (sofort actionable) |
| Asymmetrie | Gelöst durch Cap auf L | Gelöst durch identische Funktion |
| Komplexität | updatePendingTouches() pro Bar | computeDynamic() bei Abfrage |
| Performance | O(pending) Mutations pro Bar | O(touches) Berechnungen bei Abfrage |
| Trading-Tauglichkeit | Lag ist Trading-Anti-Pattern | Sofortige Bewertung |

### Risiken und Mitigationen

| Risiko | Mitigation |
|--------|------------|
| T=0-Metriken allein reichen nicht für Quality | T=0 hat 60% Gewicht; Follow-Through kommt natürlich mit Alter hinzu |
| Quality ändert sich über Zeit (nicht stabil) | Gewollt: spiegelt wachsende Bestätigung wider. Stabilisiert sich nach HORIZON Bars |
| Performance: Quality pro Touch pro Abfrage berechnen | Registry hat typisch < 500 Touches/Session; HORIZON=3 Bars Lookahead ist billig |
| Volume-Daten nicht immer verfügbar | volumeRatio defaultet auf 1.0; Proximity und Follow-Through reichen als Fallback |

### Tests

1. Live-Touch (age=0): Quality basiert zu 100% auf T=0-Metriken (kein Follow-Through)
2. Touch mit age=3: Quality enthält Follow-Through-Anteil (40%)
3. Touch mit age=30: Identische Quality wie age=3 (HORIZON gecappt)
4. Seeded und Live Touch am selben Bar → identische Quality bei gleicher Abfrage-Zeit
5. Registry-Einträge sind immutable: kein Feld ändert sich nach Schreibvorgang
6. Hoher Volume-Spike am Touch-Bar → höhere T=0-Quality
7. Starker Follow-Through (Preis entfernt sich vom Level) → höhere Gesamt-Quality
8. Kein Follow-Through (Preis bleibt am Level kleben) → niedrigere Gesamt-Quality
