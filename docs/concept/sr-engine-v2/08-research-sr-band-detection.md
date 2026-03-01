# Research: S/R-Band-Erkennung — State of the Art

**Datum:** 2026-02-28
**Anlass:** Vorbereitung PriceActionTouchMap (Stufe 2)
**Quellen:** 25+ Web-Recherchen, akademische Papers, Open-Source-Libraries

---

## Executive Summary

1. **KDE (Kernel Density Estimation) ist der akademisch fundierteste Ansatz** — erzeugt kontinuierliche Wahrscheinlichkeitsdichte, Peaks = S/R-Level, Peak-Breite = natürliche Band-Breite
2. **Bounce-Scoring mit Zeitdecay ist akademisch belegt** (Chung/Bellotti 2021, arXiv:2101.07410): Mehr Bounces = höhere Wahrscheinlichkeit weiterer Bounces, aber mit abnehmendem Effekt über Zeit
3. **Agglomeratives Clustering mit merge_percent ist der pragmatischste Merge-Ansatz** — Bänder innerhalb ATR-relativer Distanz zusammenführen
4. **ATR-relative Parametrisierung ist Schlüssel zur Generalisierbarkeit** — alle professionellen Implementierungen nutzen ATR-basierte Zonen-Breiten
5. **Band-Merge-Problem löst man durch Agglomeratives Merging + Safety-Threshold** (Max Zone Height Multiplier)

---

## Methoden-Übersicht

### KDE (Kernel Density Estimation)

Glatte Wahrscheinlichkeitsdichtekurve über Preisniveaus. Jeder historischer Preis trägt einen Gauß-Kern bei. Peaks = S/R-Level, Breite der Peaks = Band-Breite.

- Bandbreite via Silverman's Rule: `h = 1.06 * sigma * n^(-1/5)`
- **Vorteil:** Mathematisch fundiert, adaptiv an Volatilität
- **Nachteil:** Überglättet bei multimodalen Verteilungen, träge bei neuen Levels
- **Quellen:** LuxAlgo KDE Value Clouds, Medium/Open Crypto Market Data

### Volume Profile / Market Profile

Horizontale Volumen-Bins. High-Volume Nodes = S/R, Low-Volume Nodes = schnelle Bewegungszonen.

- POC = Bin mit max(volume), Value Area = 70% des Gesamtvolumens
- **Vorteil:** Direkte Marktstruktur-Abbildung
- **Nachteil:** Benötigt Volumen-Daten, statisch innerhalb Session

### Agglomeratives Clustering (empfohlen für Merge)

Bottom-up: Jeder Pivot = eigener Cluster → nächste Paare mergen bis Distanz > merge_percent.

- `merge_percent` ATR-relativ definierbar
- Median statt Mean für Robustheit gegen Ausreißer
- **Quelle:** day0market/support_resistance (GitHub)

### Price-Action Bounce-Scoring

Level durch Interaktionshistorie definiert und bewertet:

| Faktor | Gewichtung | Beschreibung |
|--------|-----------|--------------|
| Wick-Rejection (Ratio ≥ 2.0) | Hoch | Langer Docht, Preis schloss weg vom Level |
| Body-Close-Rejection | Mittel | Körper nahe Level, nächste Kerze weg |
| Volumen bei Rejection | Hoch | Hohes Vol = institutionelle Verteidigung |
| Recency (Decay 0.95/30min) | Variabel | Neuere Bounces wichtiger |
| Bounce-Anzahl | Exponentiell | Mehr Bounces = exponentiell höhere Konfidenz |

**Akademischer Beleg:** Chung/Bellotti 2021 (arXiv:2101.07410)

### Heatmap/Bucket-Ansätze

Preisbereich in kleine Bins, Interaktionen zählen, kontiguierliche High-Score-Bins = S/R-Zone.

- O(n) pro Bar, trivial parallelisierbar
- **Nachteil:** Bin-Grenzen können Level künstlich aufspalten
- **Quellen:** MQL5 Pattern Density Heatmap, LuxAlgo Volume Grid Heatmap

### DeepSupp (2025, State of the Art)

Multi-Head-Attention Autoencoder + DBSCAN. Score 0.554 vs. HMM 0.550 vs. Local Minima 0.507.

- **Erkenntnis:** Marginale Verbesserung rechtfertigt Komplexität nicht
- **Übertragbar:** DBSCAN als Clustering-Schritt, Multi-Faktor-Evaluation

---

## Band-Merge-Strategien

### Problem
Mehrere S/R-Bänder innerhalb 0.5% → "Level-Suppe", keine klare Struktur.

### Empfohlene Lösung: Agglomeratives Merging + Safety-Threshold

```
// Schritt 1: Merge naher Bänder
merge_threshold = ATR(14) * 0.15
for each band_pair (sorted by distance):
    if distance(band_a, band_b) < merge_threshold:
        merged.price = weighted_average(a, b, weights=touch_count)
        merged.touches = a.touches + b.touches
        merged.width = max(a.upper, b.upper) - min(a.lower, b.lower)

// Schritt 2: Safety-Threshold gegen überbreite Zonen
if merged.width > ATR(14) * 0.50:
    discard(merged) OR split_at_lowest_density(merged)
```

**Quelle:** LuxAlgo implementiert exakt dies ("Max Zone Height Multiplier").

---

## Intraday-spezifische Best Practices

| Aspekt | Daily/Weekly | Intraday (1-5 min) |
|--------|-------------|-------------------|
| Level-Persistenz | Wochen-Monate | Minuten-Stunden |
| Toleranzband | 0.5-1.0% | 0.1-0.3% (ATR-relativ) |
| Pivot-Erkennung | 5-20 Bars | 3-5 Bars |
| Decay-Rate | Langsam (Wochen) | Schnell (0.95/30min) |

### ATR-Skalierung (alle Parameter ATR-relativ)

| Parameter | Wert | Begründung |
|-----------|------|-------------|
| Touch-Tolerance | 0.10 * ATR | Eng genug für sub-0.2% Level |
| Band-Width | 0.20 * ATR | Schmale präzise Bänder |
| Merge-Distance | 0.15 * ATR | Nahe Level zusammenfassen |
| Max-Zone-Height | 0.50 * ATR | Safety-Threshold |
| Intraday-Decay | 0.95 / 30-Min-Block | ~50% nach 7h |
| Min-Touches-Display | 2 | 1 Touch = nicht signifikant |
| Wick-Rejection-Ratio | ≥ 2.0 | Standard-Qualitätsmaß |

---

## Empfehlung für PriceActionTouchMap

### Übernehmen

1. **ATR-relative Parametrisierung** für alle Schwellenwerte
2. **Agglomeratives Band-Merging + Safety-Threshold** (max 0.50 * ATR)
3. **Multi-Faktor Bounce-Scoring** (Wick-Qualität, Volumen, Recency-Decay)
4. **Exponentieller Decay** statt feste Fenster (akademisch belegt)
5. **Anker-Level-Integration** (VWAP, OR, Vortageshoch/-tief als Seeds)

### Nicht übernehmen

1. Deep Learning (DeepSupp) — zu komplex, marginale Verbesserung
2. Stochastische DGL-Modelle — überengineered
3. K-Means — braucht vorab K, schlecht für dynamische Märkte
4. Silverman's Rule unmodifiziert — überglättet multimodale Verteilungen

---

## Quellen

### Akademische Papers
- Chung/Bellotti (2021): arXiv:2101.07410 — Bounce-Wahrscheinlichkeit + Decay
- DeepSupp (2025): arXiv:2507.01971 — Attention-Autoencoder + DBSCAN
- Osler (2000): Fed Reserve NY — S/R bei Intraday-Wechselkursen
- WiserPub (2025): Volume-Weighted Potential Functions für S/R-Zonen

### Open-Source
- day0market/support_resistance (GitHub) — AgglomerativeClustering + TouchScorer
- ta4j 0.22+ — PriceClusterSupportIndicator, BounceCountSupportIndicator
- LuxAlgo S/R Zones Strength Classifier (TradingView/Pine Script)
