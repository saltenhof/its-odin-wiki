# ODIN-064: PriceActionTouchMap -- Level-unabhaengige S/R-Erkennung

**Modul:** odin-brain (Hauptmodul), odin-api (LevelSource-Enum)
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** Keine (orthogonal zum 6-Phasen-Plan; bestehendes Touch-Seeding aus Phase 2 bleibt unveraendert)
**Story-ID:** ODIN-064

---

## User Story

**Als** Trading-Agent
**moechte ich** S/R-Levels erkennen die durch wiederholte Price-Action-Rejections entstehen -- auch wenn DBSCAN keinen Cluster bildet --
**damit** empirisch bestaetigte Support/Resistance-Niveaus wie der IREN-$40.97-Support (4x getestet, nie durchbrochen) zuverlaessig erkannt und fuer Entry-Timing, Stop-Placement und Pyramiding genutzt werden koennen.

---

## Kontext und Motivation

### Das Problem

Bei IREN 2026-02-23 existiert ein offensichtlicher Support bei ~$40.97, der VIERMAL getestet und nie durchbrochen wurde:
- 17:15 CET: 1. Bounce
- 17:58 CET: 2. Bounce
- 18:37 CET: 3. Bounce (optimaler Einstieg, 2 Bounces bereits bestaetigt)
- 18:51 CET: 4. Abweisung -- Bears geben auf -- IREN explodiert +4.4% bis Close

Die S/R-Engine erkennt dieses Level nicht. Stattdessen gibt sie $41.07 (SWING_CLUSTER) aus. Die Distanz $40.97 <-> $41.07 = $0.10 liegt innerhalb des NMS-suppressionRadius von $0.135, sodass $40.97 zugunsten von $41.07 unterdrueckt wird.

### Warum Parameter-Tuning nicht hilft

- DBSCAN-epsilon-Aenderungen verschieben nur das Level-Set, loesen das strukturelle Problem nicht
- NMS-Sortierung wurde bereits auf touchCount-first geaendert (Stufe 1 des Sparring-Ergebnisses) -- hilft nur wenn DBSCAN das Level als Cluster erkennt
- Das Grundproblem: DBSCAN erkennt Levels durch **Pivot-Dichte** (wie viele Pivots liegen nahe beieinander?). Ein starkes Support-Level braucht aber **Bounce-Qualitaet** (wie oft hat der Preis dieses Niveau getestet und respektiert?). Beide Informationstypen sind orthogonal.

### Warum ein neuer Level-Generator noetig ist

Der bestehende DBSCAN-basierte Ansatz hat ein strukturelles Defizit: Er misst Cluster-Staerke, nicht Bounce-Qualitaet. Es gibt dokumentierte Faelle wo DBSCAN das Level gar nicht als Cluster erkennt (Streuung > epsilon, minSamples-Versagen). Fuer diese Faelle braucht es einen **unabhaengigen, level-freien Kandidaten-Generator**, der direkt aus der Price-Action ableitet wo sich Levels befinden.

---

## Architekturentscheidungen

### A1: Verhaeltnis zur bestehenden TouchRegistry / BounceDetector

**Entscheidung: Komplementaer -- beide Systeme bleiben bestehen, mit klarer Aufgabentrennung.**

| Komponente | Aufgabe | Bleibt? |
|------------|---------|---------|
| `TouchRegistry` | Spatial Index aller Touch-Events, keyed by price bin. Wird von Levels abgefragt. | Ja, unveraendert |
| `BounceDetector` | Erkennt Bounce-Events an **bestehenden** SrZones (Consumer der Pipeline-Output). Erzeugt `BounceEvent`s fuer Rules Engine. | Ja, unveraendert |
| `PriceActionTouchMap` (NEU) | Level-unabhaengiger Kandidaten-Generator. Arbeitet direkt auf Bars, nicht auf existierende Levels. Erzeugt `SrLevel`s mit Source `PRICE_ACTION`. | Neu |

**Begruendung:**
- `TouchRegistry` und `BounceDetector` sind **Konsumenten** bestehender Levels -- sie koennen keine neuen Levels erzeugen (Huehner-Ei-Problem)
- `PriceActionTouchMap` ist ein **Produzent** neuer Level-Kandidaten -- sie braucht keine bestehenden Levels als Input
- Kein Code wird geloescht oder ersetzt. Die PriceActionTouchMap fuellt die Luecke die DBSCAN laesst
- Die TouchRegistry bleibt die Single Source of Truth fuer Touch-Events. Wenn ein PRICE_ACTION-Level in die `activeLevels` aufgenommen wird, werden seine Touches ganz normal ueber die bestehende `detectTouches()`-Methode in die TouchRegistry geschrieben

### A2: Integration in die bestehende Pipeline

**Entscheidung: Parallel zu DBSCAN als eigenstaendiger Kandidaten-Generator, Fusion VOR NMS.**

Aktuelle Pipeline (Schritt 4 in `onSnapshot()`):
```
4. detectAndClusterSwingLevels()  --> DBSCAN-Cluster --> activeLevels
```

Neue Pipeline (Schritt 4a eingefuegt):
```
4.  detectAndClusterSwingLevels()    --> DBSCAN-Cluster --> activeLevels
4a. promoteFromPriceActionMap()      --> PriceAction-Levels --> activeLevels (wenn noch nicht vorhanden)
5.  detectTouches()                  --> TouchRegistry (fuer ALLE activeLevels, inkl. PRICE_ACTION)
```

**Ablauf der Integration:**

1. `PriceActionTouchMap.onBar(bar)` wird bei jedem neuen 1m-Bar aufgerufen (analog zur Touch-Detection -- NICHT interval-gated)
2. Die Map akkumuliert Rejection-Events in ATR-skalierten Preis-Buckets
3. Buckets deren `rejectScore` den Promotion-Schwellenwert ueberschreiten werden als Kandidaten gemeldet
4. `promoteFromPriceActionMap()` in `SrLevelEngine` prueft:
   - Gibt es bereits ein `SrLevel` in `activeLevels` innerhalb `fusionRadius` des Kandidaten? Wenn ja: Skip (kein Duplikat)
   - Wenn nein: Neues `SrLevel` mit Source `PRICE_ACTION` erstellen und in `activeLevels` aufnehmen
5. Das neue Level nimmt ab sofort am normalen Lifecycle teil: Touch-Detection, Scoring, NMS, OutputPolicy

**Keine separate NMS-Phase, kein Parallel-Scoring.** PRICE_ACTION-Levels werden wie SWING_CLUSTER-Levels behandelt -- sie laufen durch dieselbe Scoring- und NMS-Pipeline. Die einzige Differenz ist der Prior (0.55 statt 0.60) und die Source-Kategorie.

### A3: Neue Source-Kategorie `PRICE_ACTION`

**In `SrLevelSource` (odin-brain):**

```java
/** Price-action level detected by PriceActionTouchMap rejection scoring. */
PRICE_ACTION(0.55, "PRICE_ACTION")
```

- **Prior 0.55:** Bewusst etwas niedriger als SWING_CLUSTER (0.60), weil PRICE_ACTION-Levels kein Cluster-Minimum erfuellen. Sie muessen sich ueber Behavioral-Scoring (Touches, Quality, Recency) beweisen.
- **Eigene Kategorie "PRICE_ACTION":** Verhindert dass ein PRICE_ACTION-Level und ein SWING_CLUSTER am selben Preis als "zwei Sources" gezaehlt werden (Confluence-Inflation). In der Praxis wird die NMS-Duplikat-Pruefung in `promoteFromPriceActionMap()` das ohnehin verhindern.

### A4: Parameter der PriceActionTouchMap

Alle Parameter ATR-relativ und in `SrProperties` als neue Untergruppe `PriceActionProperties`:

| Parameter | Default | Begruendung |
|-----------|---------|-------------|
| `bucketSizeAtrFactor` | 0.10 | Bucket-Breite = 0.10 * ATR_5m. Eng genug fuer sub-0.2% Level (Research: Touch-Tolerance 0.10 * ATR) |
| `rejectionBandAtrFactor` | 0.15 | Preis muss innerhalb 0.15 * ATR_5m des Bucket-Centers liegen um als Rejection zu zaehlen |
| `recencyDecayPerBar` | 0.985 | Exponentieller Decay pro 1m-Bar. Half-Life ~46 Bars (~46 Min). Research: 0.95/30min |
| `promotionThreshold` | 1.80 | Minimum rejectScore fuer Promotion (ca. 2 frische Rejections mit vollem Gewicht) |
| `fusionRadiusAtrFactor` | 0.20 | Kandidat wird nicht promoted wenn innerhalb 0.20 * ATR_5m eines bestehenden activeLevels liegt |
| `scanRangeAtrFactor` | 4.0 | Nur Buckets innerhalb currentPrice +/- 4.0 * ATR_5m werden gescannt (Performance-Begrenzung) |
| `minBarAge` | 5 | Minimum Alter in Bars bevor ein Bucket promotet werden kann (Anti-Noise: verhindert Sofort-Promotion bei einem einzigen Spike) |
| `maxBuckets` | 200 | Maximum Anzahl aktiver Buckets (Memory-Begrenzung). Aelteste Buckets (niedrigster Score) werden evicted wenn Limit erreicht |

### A5: Band-Merge-Problem (Agglomeratives Merging)

**Entscheidung: Kein separates Band-Merging in der PriceActionTouchMap.**

**Begruendung:** Die PriceActionTouchMap erzeugt einzelne Level-Kandidaten (Punkt-Level), keine Baender. Wenn zwei nahe Buckets jeweils den Promotion-Threshold erreichen, sorgt die `fusionRadius`-Pruefung in `promoteFromPriceActionMap()` dafuer, dass nur der staerkere Kandidat promoted wird. Danach greift die bestehende NMS-Pipeline. Ein separates Agglomeratives Merging waere Over-Engineering an dieser Stelle.

Das Agglomerative Merging + Safety-Threshold aus dem Research (0.15 * ATR merge, 0.50 * ATR max zone height) bleibt als Option fuer eine spaetere Phase (wenn die PriceActionTouchMap Baender statt Punkt-Levels erzeugt).

---

## Scope

### In Scope

- `PriceActionTouchMap` Klasse (odin-brain, `de.its.odin.brain.sr`)
- `PRICE_ACTION` Enum-Wert in `SrLevelSource` (odin-brain)
- `PriceActionProperties` Record in `SrProperties` (odin-brain)
- Properties-Defaults in `odin-brain.properties`
- Integration in `SrLevelEngine.onSnapshot()` (Schritt 4a)
- Methode `promoteFromPriceActionMap()` in `SrLevelEngine`
- Unit-Tests fuer `PriceActionTouchMap`
- Unit-Tests fuer die Integration in `SrLevelEngine`
- Integrationstest: IREN-23.02-Szenario mit 4 Bounces am $40.97-Support
- Anti-Overfitting-Validierung (SPY Range-Day, Trend-Day, News-Spike)

### Out of Scope

- Aenderungen an `TouchRegistry`, `BounceDetector`, `TouchQuality`
- Aenderungen an der DBSCAN-Pipeline (Phase 1-4 des 6-Phasen-Plans)
- Aenderungen an NMS-Sortierung (bereits in Stufe 1 erledigt)
- Aenderungen an OutputQualityPolicy oder ConsolidationBandDetector
- Frontend-Aenderungen (PRICE_ACTION wird wie SWING_CLUSTER gerendert)
- Volume-basierte Rejection-Gewichtung (spaetere Verbesserung)
- Agglomeratives Band-Merging (spaetere Phase)
- Flyway-Migration (kein DB-Zugriff)

---

## Akzeptanzkriterien

### AK-1: PriceActionTouchMap Klasse

- [ ] Klasse `PriceActionTouchMap` in `de.its.odin.brain.sr` existiert
- [ ] Methode `onBar(Bar bar, double atr5m, double currentPrice)` akkumuliert Rejection-Events in Buckets
- [ ] Buckets sind ATR-skaliert (`bucketSize = bucketSizeAtrFactor * atr5m`)
- [ ] Rejection-Erkennung: Bar-Low innerhalb `rejectionBandAtrFactor * atr5m` eines Bucket-Centers UND Close oberhalb des Bucket-Centers (Support-Rejection). Analog fuer Resistance (Bar-High nahe Bucket, Close unterhalb)
- [ ] Exponentieller Recency-Decay auf alle Bucket-Scores bei jedem `onBar()`-Aufruf
- [ ] Methode `getCandidates(double atr5m)` liefert alle Buckets mit `rejectScore >= promotionThreshold` und `age >= minBarAge`
- [ ] Scan-Range begrenzt auf `currentPrice +/- scanRangeAtrFactor * atr5m`
- [ ] Bucket-Eviction bei Ueberschreitung von `maxBuckets` (niedrigster Score wird entfernt)
- [ ] Methode `reset()` loescht alle Buckets (neuer Handelstag)

### AK-2: SrLevelSource Erweiterung

- [ ] Neuer Enum-Wert `PRICE_ACTION(0.55, "PRICE_ACTION")` in `SrLevelSource`
- [ ] Eigene Kategorie "PRICE_ACTION" (nicht "CLUSTER")
- [ ] Prior 0.55 (niedriger als SWING_CLUSTER 0.60)

### AK-3: SrProperties Erweiterung

- [ ] Neuer Record `PriceActionProperties` mit allen 8 Parametern (A4)
- [ ] `SrProperties` erhaelt neues Feld `@Valid PriceActionProperties priceAction`
- [ ] Defaults in `odin-brain.properties` unter `odin.brain.sr.price-action.*`
- [ ] `@Validated` Constraints auf allen Feldern (passende `@DecimalMin`, `@Min`)

### AK-4: Integration in SrLevelEngine

- [ ] `PriceActionTouchMap` wird im Konstruktor von `SrLevelEngine` instanziiert
- [ ] `PriceActionTouchMap.onBar()` wird bei jedem neuen 1m-Bar aufgerufen (NICHT interval-gated)
- [ ] Neue Methode `promoteFromPriceActionMap(double atr5m, double band, Instant marketTime)` in `SrLevelEngine`
- [ ] Promotion-Logik: Kandidat wird nur promoted wenn kein bestehendes `SrLevel` innerhalb `fusionRadiusAtrFactor * atr5m` existiert
- [ ] Promoted Level erhaelt Source `PRICE_ACTION`, Center = Bucket-Center, Band = aktuelle `band` (aus ToleranceBand)
- [ ] `promoteFromPriceActionMap()` wird nach `detectAndClusterSwingLevels()` aufgerufen (Schritt 4a)
- [ ] Bei `reset()` wird auch `PriceActionTouchMap.reset()` aufgerufen

### AK-5: NMS-Kompatibilitaet

- [ ] PRICE_ACTION-Levels durchlaufen die bestehende NMS-Pipeline (`suppressOverdenseZones`)
- [ ] PRICE_ACTION-Levels sind `isClusterOnly()` = true (werden von NMS behandelt, nicht durchgewunken)
- [ ] Wenn ein PRICE_ACTION-Level und ein SWING_CLUSTER am selben Preis existieren, ueberlebt das Level mit mehr validierten Touches (bestehende NMS-Logik)

### AK-6: Anti-Overfitting

- [ ] Alle Parameter ATR-relativ (kein hardcodierter Preis, kein fester Abstandswert)
- [ ] Kein IREN-spezifischer Parameter
- [ ] Rejection-Definition ist universell: "Preis naehert sich Level, schliesst davon weg" (funktioniert bei SPY, TSLA, NVDA)
- [ ] Exponentieller Decay statt festes Fenster (akademisch belegt, Chung/Bellotti 2021)
- [ ] `minBarAge` verhindert Sofort-Promotion durch einzelne Spikes

---

## Technische Details

### Neue Dateien

| Datei | Package | Beschreibung |
|-------|---------|-------------|
| `PriceActionTouchMap.java` | `de.its.odin.brain.sr` | Bucket-basierter Rejection-Akkumulator |
| `PriceActionTouchMapTest.java` | `de.its.odin.brain.sr` | Unit-Tests |
| `PriceActionTouchMapIntegrationTest.java` | `de.its.odin.brain.sr` | Integrationstest mit SrLevelEngine |

### Aenderungen an bestehenden Dateien

| Datei | Aenderung |
|-------|----------|
| `SrLevelSource.java` | Neuer Enum-Wert `PRICE_ACTION(0.55, "PRICE_ACTION")` |
| `SrProperties.java` | Neues Feld `PriceActionProperties priceAction` + neuer Record |
| `SrLevelEngine.java` | `PriceActionTouchMap`-Feld, Aufruf in `onSnapshot()`, Methode `promoteFromPriceActionMap()`, `isClusterOnly()` um PRICE_ACTION erweitern |
| `odin-brain.properties` | 8 neue Properties unter `odin.brain.sr.price-action.*` |
| `TestBrainProperties.java` | `PriceActionProperties` mit Test-Defaults |

### Klasse `PriceActionTouchMap` -- Entwurf

```java
public class PriceActionTouchMap {

    /**
     * A price bucket that accumulates rejection events with recency decay.
     */
    record Bucket(double center, double rejectScore, int barsSinceCreation,
                  int totalRejections) {}

    private final SrProperties.PriceActionProperties config;
    private final NavigableMap<Integer, MutableBucket> buckets; // binKey -> bucket
    private int barsProcessed;

    /**
     * Processes a new 1-minute bar: decay all scores, detect rejections, update buckets.
     */
    public void onBar(Bar bar, double atr5m, double currentPrice) { ... }

    /**
     * Returns promotion candidates: buckets with score >= threshold and age >= minBarAge.
     */
    public List<Bucket> getCandidates(double atr5m) { ... }

    /**
     * Resets all state for a new trading day.
     */
    public void reset() { ... }
}
```

**Rejection-Erkennung (pro Bar):**
```
bucketSize = config.bucketSizeAtrFactor() * atr5m
rejectionBand = config.rejectionBandAtrFactor() * atr5m

// Support-Rejection: Bar-Low nahe einem Bucket, Close darueber
for each active bucket B within scanRange:
    if |bar.low() - B.center| <= rejectionBand AND bar.close() > B.center:
        B.rejectScore += 1.0  // Full weight for fresh rejection
        B.totalRejections++

// Resistance-Rejection: Bar-High nahe einem Bucket, Close darunter
for each active bucket B within scanRange:
    if |bar.high() - B.center| <= rejectionBand AND bar.close() < B.center:
        B.rejectScore += 1.0
        B.totalRejections++

// Neuer Bucket fuer ungedeckte Rejections (bar.low nicht nahe einem Bucket):
if no bucket covers bar.low():
    create new bucket at bar.low(), rejectScore = 1.0

// Decay alle Buckets:
for each bucket B:
    B.rejectScore *= config.recencyDecayPerBar()
```

**Bucket-Binning:**
```
binKey = Math.round(price / bucketSize)
```

Nicht `$0.01`-Bins wie die TouchRegistry, sondern ATR-skalierte Bins. Das ist bewusst grober -- die PriceActionTouchMap aggregiert ueber breitere Bereiche, waehrend die TouchRegistry exakte Preise trackt.

### Integration in `SrLevelEngine.onSnapshot()` -- Pseudocode

```java
// Schritt 4a (nach detectAndClusterSwingLevels, vor detectTouches):
if (priceActionTouchMap != null && currentBarCount > lastOneMinBarCount) {
    Bar currentBar = oneMinBars.getLast();
    priceActionTouchMap.onBar(currentBar, atr5m, currentPrice);
    promoteFromPriceActionMap(atr5m, band, marketTime);
}
```

```java
private void promoteFromPriceActionMap(double atr5m, double band, Instant marketTime) {
    double fusionRadius = config.priceAction().fusionRadiusAtrFactor() * atr5m;
    List<PriceActionTouchMap.Bucket> candidates = priceActionTouchMap.getCandidates(atr5m);

    for (PriceActionTouchMap.Bucket candidate : candidates) {
        boolean alreadyCovered = false;
        for (SrLevel existing : activeLevels) {
            if (Math.abs(existing.center() - candidate.center()) <= fusionRadius) {
                alreadyCovered = true;
                break;
            }
        }
        if (!alreadyCovered) {
            SrLevel newLevel = new SrLevel(
                SrLevelSource.PRICE_ACTION, candidate.center(), band, marketTime, touchRegistry);
            activeLevels.add(newLevel);
            LOG.info("PriceAction: promoted level at {} (rejectScore={}, rejections={})",
                String.format("%.4f", candidate.center()),
                String.format("%.2f", candidate.rejectScore()),
                candidate.totalRejections());
        }
    }
}
```

### Properties-Defaults (`odin-brain.properties`)

```properties
# PriceActionTouchMap (ODIN-064)
odin.brain.sr.price-action.bucket-size-atr-factor=0.10
odin.brain.sr.price-action.rejection-band-atr-factor=0.15
odin.brain.sr.price-action.recency-decay-per-bar=0.985
odin.brain.sr.price-action.promotion-threshold=1.80
odin.brain.sr.price-action.fusion-radius-atr-factor=0.20
odin.brain.sr.price-action.scan-range-atr-factor=4.0
odin.brain.sr.price-action.min-bar-age=5
odin.brain.sr.price-action.max-buckets=200
```

---

## Test-Strategie

### Unit-Tests (`PriceActionTouchMapTest`)

| Test | Beschreibung |
|------|-------------|
| `singleRejection_incrementsScore` | Ein Bar mit Low nahe Bucket-Center und Close darueber erhoet rejectScore um 1.0 |
| `multipleRejections_accumulateScore` | 3 Rejections am selben Bucket ergeben rejectScore ~3.0 (vor Decay) |
| `recencyDecay_reducesScoreOverTime` | Nach N Bars ohne Rejection sinkt der Score gemaess Decay-Formel |
| `promotionThreshold_correctlyFilters` | Nur Buckets mit score >= 1.80 werden als Kandidaten geliefert |
| `minBarAge_preventsImmediatePromotion` | Bucket mit hohem Score aber age < minBarAge wird nicht promoted |
| `scanRange_limitsBucketCreation` | Buckets ausserhalb `currentPrice +/- scanRange * ATR` werden nicht erstellt |
| `maxBuckets_evictsLowestScore` | Bei Ueberschreitung von maxBuckets wird der Bucket mit dem niedrigsten Score entfernt |
| `reset_clearsAllState` | Nach `reset()` sind alle Buckets weg, barsProcessed = 0 |
| `resistanceRejection_detected` | Bar-High nahe Bucket + Close darunter wird als Resistance-Rejection erkannt |
| `noRejection_whenCloseDoesNotBounce` | Bar-Low nahe Bucket aber Close UNTER Bucket-Center ist KEINE Support-Rejection |
| `atrScaling_bucketSizeAdapts` | Bei ATR 0.30 ist bucketSize 0.03, bei ATR 0.60 ist bucketSize 0.06 |

### Unit-Tests (`SrLevelEngine` Erweiterung)

| Test | Beschreibung |
|------|-------------|
| `priceActionPromotion_createsNewLevel` | PriceActionTouchMap-Kandidat wird als PRICE_ACTION-Level in activeLevels aufgenommen |
| `priceActionPromotion_skipsDuplicate` | Kandidat innerhalb fusionRadius eines bestehenden Levels wird nicht promoted |
| `priceActionLevel_survivesNms` | Ein PRICE_ACTION-Level mit Touches ueberlebt NMS wenn es mehr Touches hat als ein SWING_CLUSTER |
| `priceActionLevel_droppedByOptionD` | Ein PRICE_ACTION-Level mit 0 validierten Touches wird von Option D entfernt |

### Integrationstests

| Test | Beschreibung |
|------|-------------|
| `irenBounceScenario_detectsSupport` | Replay von 4 Bounce-Bars am $40.97-Level: PriceActionTouchMap erzeugt Kandidat, Level wird promoted, touchCount >= 2 im Output-Snapshot. **Der Kerntestfall.** |
| `trendDay_noSpuriousLevels` | Bei starkem Aufwaertstrend: keine spurious PRICE_ACTION-Levels die den Trend blockieren |
| `rangeDay_detectsBothSides` | Bei Seitwaertsmarkt: Support UND Resistance PRICE_ACTION-Levels werden korrekt erkannt |

### Anti-Overfitting-Tests

| Szenario | Erwartung |
|----------|-----------|
| SPY Range-Day (seitwaerts) | Levels mit tatsaechlichen Bounces werden erkannt |
| Trend-Day (starker Move nach oben) | Keine spurious Levels die den Trend blockieren |
| News-Spike-Day (ploetzlicher Move) | Level-Set wird ohnehin durchgebrochen, PriceActionTouchMap neutral (Decay raeumt alte Buckets auf) |
| Large-Cap mit breiter ATR (z.B. TSLA $300, ATR $3.00) | ATR-Skalierung funktioniert: bucketSize=$0.30, rejectionBand=$0.45 -- angemessen fuer $300-Aktie |
| Small-Cap mit enger ATR (z.B. $5, ATR $0.10) | bucketSize=$0.01, rejectionBand=$0.015 -- praezise genug fuer $5-Aktie |

---

## Anti-Overfitting-Check

**Zentrale Frage:** Wuerde die PriceActionTouchMap auf einem anderen Instrument oder einem anderen Tag genauso sinnvoll funktionieren?

1. **ATR-relative Parametrisierung:** Alle Schwellenwerte skalieren mit der Volatilitaet des Instruments. $40-Aktie mit ATR $0.30 und $300-Aktie mit ATR $3.00 werden identisch behandelt.

2. **Rejection-Definition ist universell:** "Preis naehert sich einem Niveau, schliesst davon weg" ist das grundlegendste Price-Action-Signal. Es funktioniert bei jedem Instrument in jedem Markt.

3. **Exponentieller Decay:** Alte Rejections verlieren an Bedeutung. Kein festes Fenster das bei unterschiedlichen Handelszeiten bricht.

4. **Kein IREN-spezifischer Parameter:** Kein hardcodierter Preis, keine IREN-spezifische Bucket-Groesse, kein Fix fuer genau diesen Chartverlauf.

5. **Prior 0.55 < SWING_CLUSTER 0.60:** PRICE_ACTION-Levels muessen sich ueber Behavioral-Scoring (Touches, Quality) beweisen. Ein einmaliger Spike reicht nicht -- der `minBarAge`-Guard und der Promotion-Threshold stellen sicher dass nur wiederholte Rejections promoted werden.

6. **Frage an den Implementierer:** Nach der Implementierung folgende Gedankenexperimente durchfuehren und im `protocol.md` dokumentieren:
   - "Wuerde das bei einem Pharma-Stock mit Gap-Up und anschliessender Konsolidierung funktionieren?"
   - "Wuerde das bei einem ETF wie QQQ mit engen Spreads funktionieren?"
   - "Was passiert bei einer Aktie die den ganzen Tag nur faellt (kein Bounce)?"

---

## Konzept-Referenzen

- `docs/concept/sr-engine-v2/07-sparring-bounce-level-detection.md` -- **Gesamtes Dokument**: Problem-Analyse, Diskussion der 5 Loesungsansaetze (A-E), Empfehlung "Zweistufige Strategie" (Stufe 2 = diese Story)
  - Abschnitt "Der fundamentale Zielkonflikt": Cluster-Staerke vs. Bounce-Qualitaet
  - Abschnitt "Ansatz E (ChatGPT)": PriceActionTouchMap als eigenstaendiger Kandidaten-Generator
  - Abschnitt "Empfohlene Loesung: Stufe 2": Parameter, Fusion, Promotion-Schwelle
  - Abschnitt "Anti-Overfitting-Check": Generalisierbarkeits-Bewertung

- `docs/concept/sr-engine-v2/08-research-sr-band-detection.md` -- **Gesamtes Dokument**: State of the Art, akademische Belege
  - Abschnitt "Price-Action Bounce-Scoring": Multi-Faktor-Gewichtung, Chung/Bellotti 2021
  - Abschnitt "Heatmap/Bucket-Ansaetze": O(n) pro Bar, Bin-Grenzen-Problem
  - Abschnitt "Intraday-spezifische Best Practices": ATR-Skalierung, Decay-Rate
  - Tabelle "ATR-Skalierung": touch-tolerance 0.10*ATR, band-width 0.20*ATR

- `CLAUDE.md` -- Abschnitt "Support/Resistance Level Detection (kritischster Optimierungspunkt)": Multiple-Bounce-Regel, Praezision der Level, IREN-Sequenz

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package `de.its.odin.brain.sr`, keine Cross-Modul-Abhaengigkeit
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints, ChatGPT-Pool-Review Pflicht
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (kein `var`, keine Magic Numbers, Records fuer DTOs, JavaDoc Pflicht), Test-Konventionen (`*Test` fuer Surefire, `*IntegrationTest` fuer Failsafe)

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien (AK-1 bis AK-6)
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (Bucket ist ein Record)
- [ ] ENUM statt String fuer endliche Mengen (`PRICE_ACTION` in SrLevelSource)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.sr.price-action.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `PriceActionTouchMap` (alle 11 Tests aus der Test-Strategie)
- [ ] Unit-Tests fuer SrLevelEngine-Integration (alle 4 Tests)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs: Keine Spring-Kontexte noetig (Pro-Pipeline-Komponenten)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: IREN-Bounce-Szenario (4 Bounces, Level wird erkannt)
- [ ] Integrationstest: Trend-Day (keine spurious Levels)
- [ ] Integrationstest: Range-Day (Support + Resistance erkannt)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `PriceActionTouchMap.java`, `promoteFromPriceActionMap()`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei ATR=0? Bei ATR=NaN? Bei einem einzigen Bar? Bei 1000+ Bars? Bei Preis = 0? Bei negativem rejectScore nach viel Decay? Bei bucketSize > scanRange?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe PriceActionTouchMap und promoteFromPriceActionMap auf Bugs, Race Conditions, Null-Safety, Integer-Overflow bei Binning, Memory-Leaks bei Bucket-Accumulation, Off-by-one bei Decay"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + Sparring-Ergebnis (07) + Research (08) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche: Parameter-Werte, Rejection-Definition, Promotion-Logik, Fusion-Radius, Decay-Formel, Anti-Overfitting-Massnahmen"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen die im Konzept nicht behandelt sind? Z.B.: Was passiert bei Session-Wechsel (Pre-Market -> RTH)? Bei Gap-Open? Bei Halts? Bei Aktien mit ungewoehnlich breitem Spread?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

1. **Bucket-Binning ist ATR-skaliert, nicht fest $0.01.** Die TouchRegistry benutzt $0.01-Bins fuer exakte Touch-Preise. Die PriceActionTouchMap benutzt ATR-skalierte Bins (z.B. $0.03 bei ATR $0.30). Das ist beabsichtigt -- die Map aggregiert ueber breitere Bereiche.

2. **`isClusterOnly()` erweitern.** Die bestehende Methode in `SrLevelEngine` prueft ob alle Contributing Levels SWING_CLUSTER sind. PRICE_ACTION muss hier ebenfalls als "cluster-only" gelten, damit NMS und Option D greifen. Pruefe die Methode und erweitere sie.

3. **Kein neuer Thread, keine Asynchronitaet.** `PriceActionTouchMap.onBar()` wird synchron im Pipeline-Thread aufgerufen. Die Klasse ist ein POJO wie alle anderen Pro-Pipeline-Komponenten.

4. **Reihenfolge in `onSnapshot()` ist kritisch.** `priceActionTouchMap.onBar()` muss VOR `detectTouches()` aufgerufen werden, damit neu promotete PRICE_ACTION-Levels sofort Touch-Events sammeln koennen.

5. **ATR-NaN-Guard:** Wenn `atr5m` NaN ist (Warmup-Phase), darf `onBar()` nichts tun. Guard am Anfang der Methode.

6. **MutableBucket intern, Bucket-Record extern.** Intern arbeitet die Map mit mutablen Buckets (Score wird in-place aktualisiert). Die `getCandidates()`-Methode liefert immutable `Bucket`-Records zurueck (sauberes API-Design).

7. **TestBrainProperties erweitern.** Alle bestehenden Test-Factories fuer `SrProperties` (z.B. in `SrLevelEngineTest`, `SrLevelCalculationServiceTest`) muessen um den neuen `PriceActionProperties`-Parameter erweitert werden. Verwende die Default-Werte aus den Properties.

8. **IREN-Testdaten:** Fuer den Integrationstest synthetische Bars erstellen die das 4-Bounce-Pattern nachbilden (Lows bei ~$40.97 mit Closes darueber). Keine echten Marktdaten noetig -- das Pattern ist das was zaehlt, nicht der exakte IREN-Kursverlauf.

9. **Anti-Overfitting-Denkexperiment nach der Implementierung:** Im `protocol.md` die drei Gedankenexperimente aus dem Anti-Overfitting-Check dokumentieren. Wenn ein Szenario problematisch erscheint, im "Offene Punkte"-Abschnitt eskalieren.

10. **Performance-Budget:** `onBar()` wird bei jedem 1m-Bar aufgerufen (390 Bars pro RTH-Session). Mit maxBuckets=200 und NavigableMap-basiertem Scan ist die Performance unkritisch (< 1ms pro Aufruf). Kein Profiling noetig, aber defensive Guards gegen unbegrenztes Wachstum (maxBuckets) sind Pflicht.
