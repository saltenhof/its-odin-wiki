# S/R Level Engine v2 — Offene Probleme

**Stand:** 2026-02-27
**Bezug:** IREN 2026-02-23 (Referenz-Handelstag)
**Aktueller API-Output:** 12 Zonen, davon 5 OR, 6 CLUSTER, 1 CLUSTER+VWAP

---

## Ist-Zustand: API-Output (GET /api/v1/charts/IREN/sr-levels?tradingDate=2026-02-23)

```
#   Center   Band     Confidence  TouchCount  Sources
1   38.962   0.0337   0.298       0           OR
2   39.641   0.0337   0.266       0           OR
3   39.716   0.0337   0.266       0           OR
4   40.320   0.0337   0.446       1           OR
5   40.470   0.0337   0.358       0           OR
6   40.926   0.0337   0.288       5           CLUSTER
7   41.068   0.0337   0.412       12          CLUSTER
8   41.337   0.0420   0.565       9           CLUSTER,VWAP
9   41.703   0.0337   0.309       5           CLUSTER
10  41.815   0.0337   0.369       3           CLUSTER
11  41.929   0.0337   0.320       11          CLUSTER
12  42.073   0.0370   0.322       41          CLUSTER
```

**IREN 23.02 Handelskontext:** Stock eröffnete bei ~39.00, rallied auf ~42.50, schloss bei ~42.30.
Die relevante Handelszone lag ab ~16:00 CET bei 40.80–42.50.

---

## Problem 1: OR-Level-Überflutung

### Beschreibung

5 von 12 Zonen (42%) sind Opening-Range-Level. Drei davon (38.96, 39.64, 39.72) liegen
**2.50–3.30 USD (6–8%) unter der Haupthandelszone** und haben **0 Touches**. Sie sind
nicht actionable — kein Trader oder Algorithmus wird an $38.96 Support erwarten wenn
die Aktie seit 2 Stunden über $41 handelt.

### Ursache

Die Engine generiert 6 OR-Level pro Session (OR5_HIGH, OR5_MID, OR5_LOW, OR10_HIGH,
OR10_MID, OR10_LOW) aus den ersten 5/10 RTH-Minuten. Diese Level werden in der
`keepAlways`-Liste von `suppressOverdenseZones()` geführt — sie überleben NMS immer,
unabhängig von Relevanz, Distanz zum aktuellen Preis oder Touch-Historie.

Bei einem starken Opening-Move (IREN rallied 8% vom Open) werden die OR-Level-Preise
schnell irrelevant, belegen aber 5 von 12 maxZones-Slots und verdrängen potenziell
relevantere Cluster-Level.

### Betroffener Code

- `SrLevelEngine.java`, `updateOpeningRanges()` (Zeilen 271–329): Generiert OR5/OR10
- `SrLevelEngine.java`, `addOrUpdateStaticLevels()` (Zeilen 334–361): Erstellt 6 SrLevel
- `SrLevelEngine.java`, `suppressOverdenseZones()` (Zeilen 942–1019): `keepAlways` enthält
  ALLE Zonen mit mindestens einem statischen Level (OR, VWAP, PD) — kein Distance-Filter
- `SrLevelSource.java`: Alle OR-Quellen haben Prior=0.80, Kategorie "OR"
- `SrProperties.ConfluenceProperties`: `maxZones=12` (globales Limit)

### Konkretes Problem am IREN-Beispiel

| Zone       | Distanz zu Close (~42.30) | Actionable? |
|------------|---------------------------|-------------|
| $38.96 OR  | -7.9%                     | Nein        |
| $39.64 OR  | -6.3%                     | Nein        |
| $39.72 OR  | -6.1%                     | Nein        |
| $40.32 OR  | -4.7%                     | Grenzwertig |
| $40.47 OR  | -4.3%                     | Grenzwertig |

Keines dieser Level wurde im RTH-Verlauf je wieder angetestet (touchCount 0–1).

### Soll-Zustand

- OR-Level die weit vom aktuellen Preis entfernt sind UND keine Touches haben, sollten
  aus der Ausgabe gefiltert werden
- maxZones-Slots sollen primär von Level mit tatsächlicher Price-Action belegt werden
- Ein OR-Level das als S/R bestätigt wurde (Touches) soll erhalten bleiben

---

## Problem 2: Overdense Konsolidierungszone (41.70–42.07)

### Beschreibung

4 Cluster-Zonen liegen innerhalb einer Spanne von $0.37 (0.9%):

```
41.703  touchCount=5   confidence=0.309  CLUSTER
41.815  touchCount=3   confidence=0.369  CLUSTER
41.929  touchCount=11  confidence=0.320  CLUSTER
42.073  touchCount=41  confidence=0.322  CLUSTER
```

Das sind zu viele Level für eine Konsolidierungszone. Ein Trader braucht hier maximal
2 Level: die **Obergrenze** (~42.07) und die **Untergrenze** (~41.70) der Zone.
Die Innenstruktur (41.81, 41.93) ist Noise.

### Ursache

**NMS-Suppressions-Radius ist zu klein.** Der aktuelle Radius berechnet sich als:

```
suppressionRadius = max(1.5 × epsilon, 3.0 × baseBand)
                  = max(1.5 × 0.15 × ATR5m, 3.0 × 0.0337)
                  = max(~0.09, ~0.10)
                  ≈ $0.10
```

Die Abstände zwischen den 4 Clustern betragen 0.11–0.14, also knapp ÜBER dem
Suppressions-Radius. Dadurch überlebt jeder Cluster die NMS.

**Separate H/L-Clustering (Phase 3) hat geholfen** — von 6 auf 4 reduziert — aber
der NMS-Radius skaliert nicht mit der Konsolidierungsbreite.

### Betroffener Code

- `SrLevelEngine.java`, `suppressOverdenseZones()`: NMS-Radius (Zeile 999)
- Konstanten: `NMS_EPSILON_MULTIPLIER=1.5`, `NMS_BASE_BAND_MULTIPLIER=3.0` (Zeilen 56–59)

### Soll-Zustand

In einer Konsolidierungszone (mehrere Cluster innerhalb ~1% Preisspanne) sollen maximal
2–3 Level überleben: obere Grenze, untere Grenze, und ggf. VWAP/Mitte.

---

## Problem 3: Lookahead-Asymmetrie bei Touch-Seeding

### Beschreibung

Touch-Seeding (Phase 2) und Live-Touch-Erkennung bewerten Touch-Qualität unterschiedlich:

- **Geseedete Touches:** Der Pivot liegt in der Vergangenheit. `seedTouchesFromPivots()`
  hat Zugriff auf die **folgenden Bars nach dem Pivot** (lookahead), weil die gesamte
  Bar-Historie bereits vorliegt. TouchQuality kann Separation, Follow-Through und
  Bounce-Stärke vollständig bewerten.

- **Live-Touches:** `detectTouches()` verarbeitet den aktuellen Bar. Die Lookahead-Liste
  ist praktisch leer (oder enthält nur 1–2 Bars vom aktuellen Snapshot). TouchQuality
  fällt auf "bar-only"-Metriken zurück → systematisch niedrigere Bewertung.

### Konsequenz

Geseedete Touches erhalten systematisch höhere Quality-Werte als Live-Touches für
identische Price-Action-Muster. Das verzerrt:

1. **touchCount (validated):** Seeded Touches überspringen den quality ≥ 0.50 Threshold
   leichter → inflationierter validatedTouchCount
2. **Confidence-Scoring:** Behavioral Component gewichtet Touch-Quality mit 45% →
   Level mit vielen Seeded Touches wirken stärker als sie live wären
3. **Option D Filter:** Level mit Seeded Touches überleben den Noise-Filter,
   auch wenn ihre Live-Touch-Qualität unter 0.50 läge

### Betroffener Code

- `SrLevelEngine.java`, `seedTouchesFromPivots()` (Zeilen 568–618): Baut Lookahead-Liste
  aus `oneMinBars.subList(absIdx+1, absIdx+1+lookaheadCount)`
- `SrLevelEngine.java`, `detectTouches()`: Baut Lookahead aus aktuellen Bars
  (typisch 0–2 Bars verfügbar)
- `TouchQuality.java`, `computeForSupport()`/`computeForResistance()`: Lookahead-abhängige
  Quality-Komponenten (Separation, Follow-Through)

### Soll-Zustand

Touch-Seeding und Live-Touch-Erkennung sollen vergleichbare Quality-Bewertungen produzieren.
Entweder:
- (a) Seeding ohne Lookahead (bar-only-Metriken) → konsistent mit Live
- (b) Deferred Quality: Live-Touches werden nach N Bars nachbewertet wenn Lookahead vorliegt
- (c) Lookahead-Penalty: Seeded Touches bekommen einen Abschlag auf die Quality

---

## Problem 4: Reconciliation-Drift (Touch-Historie-Verlust)

### Beschreibung

`reconcileSwingClusters()` matched neue DBSCAN-Cluster gegen bestehende SWING_CLUSTER-Level
basierend auf Center-Distanz < epsilon. Wenn der Cluster-Center durch neue Pivots oder
geändertes Gewicht über epsilon driftet, wird:

1. Das **alte Level gelöscht** (inkl. seiner gesamten Touch-Historie)
2. Ein **neues Level erstellt** (nur mit Seeded Touches aus dem neuen Cluster)

Alle **Live-Touches** die zwischen dem ersten und dem letzten Clustering zum alten Level
hinzugefügt wurden, gehen verloren.

### Konkretes Risiko

- Szenario: Cluster bei $41.80 mit 5 Live-Touches. Neuer Pivot verschiebt den
  DBSCAN-Center auf $41.95 (Drift > epsilon ≈ $0.09). Altes Level bei $41.80 wird
  gelöscht, neues Level bei $41.95 startet mit 0 Live-Touches + Seeded Touches.
- Die 5 echten Live-Touches am $41.80-Level sind unwiederbringlich verloren.

### Betroffener Code

- `SrLevelEngine.java`, `reconcileSwingClusters()` (Zeilen 437–567): Matching-Logik
  basierend auf `Math.abs(newCenter - existingCenter) < epsilon`
- Kein Grace-Period, kein Fallback-Matching, kein Touch-Transfer

### Häufigkeit

In Praxis selten bei stabilen Trends. Kritisch bei:
- Seitwärtsmärkten (Cluster-Center oszilliert)
- Starken Trend-Reversals (neue Pivots verschieben Center deutlich)
- Kleinem epsilon (enger Toleranzbereich)

### Soll-Zustand

Touch-Historie darf nicht durch Cluster-Drift verloren gehen. Optionen:
- Grace Period (unmatched Level N Zyklen behalten)
- Band-basiertes Matching statt Center-Distanz
- Touch-Transfer vom alten zum neuen Level bei Drift

---

## Problem 5: Gesamtqualität der Zone-Ausgabe

### Beschreibung

Das übergreifende Problem: Die aktuelle Ausgabe enthält zu viele low-value Zonen.
Von 12 Zonen haben:

- **5 Zonen** Confidence < 0.30 (noise-level)
- **3 Zonen** Confidence 0.30–0.40 (marginal)
- **3 Zonen** Confidence 0.40–0.50 (moderat)
- **1 Zone** Confidence > 0.50 (die VWAP-Zone bei 41.34)

Die höchste Confidence im gesamten Output ist **0.565** — das ist niedrig. Ein Chart
mit 12 Linien, von denen die meisten unter 35% Confidence liegen, ist nicht actionable.

### Zusammenhang mit den anderen Problemen

Dieses Problem ist die **Folge** der anderen vier:
- Problem 1 (OR-Überflutung): 5 low-value OR-Zonen verdrängen potenziell bessere Cluster
- Problem 2 (Overdense Konsolidierung): 4 Cluster-Zonen teilen sich die Touches einer Zone
- Problem 3 (Lookahead-Asymmetrie): Confidence-Werte sind verzerrt
- Problem 4 (Drift-Verlust): Touch-Verlust drückt Confidence etablierter Level

### Soll-Zustand

- **Weniger aber stärkere Zonen:** 5–8 Zonen statt 12, mit höherer durchschnittlicher Confidence
- **Minimum-Confidence-Threshold:** Zonen unter einem Schwellwert (z.B. 0.30) nicht ausgeben
- **OR-Level nur wenn relevant:** OR-Level nur in Ausgabe wenn sie Touches haben oder
  nah am aktuellen Preis liegen
- **Konsolidierung als Zone:** Statt 4 einzelner Cluster eine zusammengefasste
  Konsolidierungszone mit allen aggregierten Touches

---

## Abhängigkeiten zwischen den Problemen

```
Problem 1 (OR-Überflutung) ────────────────> Problem 5 (Gesamtqualität)
Problem 2 (Overdense Konsolidierung) ──────> Problem 5
Problem 3 (Lookahead-Asymmetrie) ──────────> Problem 5 (indirekt)
Problem 4 (Reconciliation-Drift) ─────────> Problem 5 (indirekt)

Problem 1 ist unabhängig von 2, 3, 4
Problem 2 ist unabhängig von 1, 3, 4
Problem 3 ist unabhängig von 1, 2, 4
Problem 4 ist unabhängig von 1, 2, 3
Problem 5 löst sich (teilweise) durch Lösung von 1–4
```

Empfohlene Bearbeitungsreihenfolge:
1. Problem 1 (OR-Überflutung) — größter sichtbarer Impact
2. Problem 2 (Overdense Konsolidierung) — zweitgrößter Impact
3. Problem 5 (Gesamtqualität) — Confidence-Threshold + maxZones-Reduktion
4. Problem 3 (Lookahead-Asymmetrie) — Quality-Korrektur
5. Problem 4 (Reconciliation-Drift) — Edge-Case-Absicherung

---

## Referenzdateien

| Datei | Pfad |
|-------|------|
| SrLevelEngine | `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelEngine.java` |
| SrDbscan | `odin-brain/src/main/java/de/its/odin/brain/sr/SrDbscan.java` |
| PriceCluster | `odin-brain/src/main/java/de/its/odin/brain/sr/PriceCluster.java` |
| SrLevel | `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevel.java` |
| SrZone | `odin-brain/src/main/java/de/its/odin/brain/sr/SrZone.java` |
| SrLevelSource | `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelSource.java` |
| TouchQuality | `odin-brain/src/main/java/de/its/odin/brain/sr/TouchQuality.java` |
| TouchEvent | `odin-brain/src/main/java/de/its/odin/brain/sr/TouchEvent.java` |
| SupportLevelConfidence | `odin-brain/src/main/java/de/its/odin/brain/sr/SupportLevelConfidence.java` |
| SrProperties | `odin-brain/src/main/java/de/its/odin/brain/sr/SrProperties.java` |
| SrLevelCalculationService | `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelCalculationService.java` |
| SrQueryService | `odin-app/src/main/java/de/its/odin/app/service/SrQueryService.java` |
| ToleranceBand | `odin-brain/src/main/java/de/its/odin/brain/sr/ToleranceBand.java` |
| PivotDetector | `odin-brain/src/main/java/de/its/odin/brain/sr/PivotDetector.java` |

Alle Pfade relativ zu: `T:/codebase/its_odin/its-odin-backend/`
