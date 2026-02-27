# Mini-Konzept 6: Band-API und Hybrid-Rendering

**Kontext:** Konsolidierungsbänder (Konzept 2) müssen als Bereiche kommuniziert werden, nicht als Linien. Reguläre S/R-Level erhalten einen dezenten Halo.
**Rendering-Entscheidung:** Hybrid — breite Bänder für Konsolidierungen, Linie + Halo für scharfe Level.

---

## API-Änderung: SrZoneDto erweitern

### Aktuelles DTO

```java
record SrZoneDto(
    double center,
    double band,
    double confidence,
    int touchCount,
    String sources
)
```

### Neues DTO

```java
record SrZoneDto(
    double center,          // Level-Preis (bei Band: POC)
    double band,            // Toleranzband (bei Band: halbe Breite)
    double confidence,
    int touchCount,
    String sources,         // "OR", "CLUSTER", "CLUSTER,VWAP", "CONSOLIDATION"
    String zoneType         // "LEVEL" oder "BAND"
) {}
```

**Neues Feld `zoneType`:**

| Wert | Bedeutung | UI-Rendering |
|------|-----------|--------------|
| `"LEVEL"` | Scharfes S/R-Level (OR, VWAP, einzelner Cluster) | Linie + dezenter ±band Halo |
| `"BAND"` | Konsolidierungszone (breiter Preisbereich mit hoher Touch-Dichte) | Gefülltes halbtransparentes Rechteck |

**Warum reicht `band` als Breiten-Information:**
- Bei `LEVEL`: `band` ist klein (~$0.03–0.05), Halo ist schmal
- Bei `BAND`: `band` ist groß (~$0.15–0.25), gefüllter Bereich ist breit
- Die Unterscheidung `LEVEL` vs `BAND` steuert das Rendering, `band` steuert die Breite

### REST-Endpoint (unverändert, nur DTO erweitert)

```
GET /api/v1/charts/{symbol}/sr-levels?tradingDate={date}
```

Response-Beispiel:
```json
[
  {
    "center": 41.067,
    "band": 0.034,
    "confidence": 0.71,
    "touchCount": 12,
    "sources": "CLUSTER",
    "zoneType": "LEVEL"
  },
  {
    "center": 41.337,
    "band": 0.042,
    "confidence": 0.90,
    "touchCount": 9,
    "sources": "CLUSTER,VWAP",
    "zoneType": "LEVEL"
  },
  {
    "center": 42.04,
    "band": 0.225,
    "confidence": 0.65,
    "touchCount": 60,
    "sources": "CONSOLIDATION",
    "zoneType": "BAND"
  }
]
```

---

## Frontend: Hybrid-Rendering

### LEVEL-Rendering (Linie + Halo)

```
Linie:    center-Preis, horizontale Linie über Chart-Breite
          Farbe: basierend auf Confidence (Opacity-Mapping)
          Strichstärke: 1.5px (normal) / 2px (hohe Confidence)

Halo:     ±band um center, halbtransparentes Rechteck
          Farbe: gleiche Farbe wie Linie, Opacity = 0.08–0.12
          Zweck: visualisiert die Toleranzzone dezent

Label:    Preis + Source-Abkürzung rechts am Chart-Rand
```

**Beispiel:** VWAP bei $41.34, band=0.042
→ Scharfe Linie bei $41.34, dezenter Schimmer $41.30–$41.38

### BAND-Rendering (Gefüllter Bereich)

```
Bereich:  [center - band, center + band], gefülltes Rechteck
          Farbe: dedizierte Konsolidierungsfarbe
          Opacity: 0.12–0.18 (deutlich sichtbar aber nicht dominant)
          Rand: gestrichelte Linie oben und unten (Breakout-Grenzen)

POC:      Dünne Linie bei center (POC = Point of Control)
          Farbe: gleiche wie Bereich, aber volle Opacity
          Strichstärke: 1px, gestrichelt

Label:    "Consol. $41.65–$42.10" rechts am Chart-Rand
```

**Beispiel:** Konsolidierung $41.65–$42.10, POC bei $42.04
→ Halbtransparentes Rechteck über den ganzen Bereich, gestrichelter oberer/unterer
  Rand, dünne gestrichelte POC-Linie bei $42.04

### Farben (basierend auf User-Vorgaben)

| Element | Farbe | Opacity |
|---------|-------|---------|
| Support-Linie (LEVEL) | `#0EB35B` (Bullish Green) | Confidence-abhängig (35–80%) |
| Resistance-Linie (LEVEL) | `#FC3243` (Bearish Red) | Confidence-abhängig (35–80%) |
| Support-Halo (LEVEL) | `#0EB35B` | 8–12% |
| Resistance-Halo (LEVEL) | `#FC3243` | 8–12% |
| Konsolidierungsband (BAND) | `#FFA726` (Orange/Amber) | 12–18% |
| Konsolidierungs-Rand (BAND) | `#FFA726` | 40%, gestrichelt |
| Konsolidierungs-POC (BAND) | `#FFA726` | 60%, gestrichelt |
| VWAP-Linie | `#2196F3` (wie bisher) | Confidence-abhängig |

**Orange/Amber für Konsolidierung:** Signalisiert "Warnung/Neutral" — weder bullish
(grün) noch bearish (rot). Konsistent mit der Semantik "Chop-Zone, kein klares Signal".

### TypeScript-Interface

```typescript
type SrZoneType = 'LEVEL' | 'BAND';

interface SrZone {
  center: number;
  band: number;
  confidence: number;
  touchCount: number;
  sources: string;
  zoneType: SrZoneType;
}
```

### Chart-Rendering-Logik (Pseudocode)

```typescript
function renderSrZones(zones: SrZone[], currentPrice: number) {
  for (const zone of zones) {
    const side = zone.center < currentPrice ? 'support' : 'resistance';

    if (zone.zoneType === 'BAND') {
      // Konsolidierungsband
      renderFilledRect(
        zone.center - zone.band,   // bottom
        zone.center + zone.band,   // top
        CONSOLIDATION_COLOR,
        CONSOLIDATION_OPACITY
      );
      renderDashedLine(zone.center - zone.band, CONSOLIDATION_COLOR, 0.4);  // bottom edge
      renderDashedLine(zone.center + zone.band, CONSOLIDATION_COLOR, 0.4);  // top edge
      renderDashedLine(zone.center, CONSOLIDATION_COLOR, 0.6);             // POC
      renderLabel(`Consol. ${formatPrice(zone.center - zone.band)}–${formatPrice(zone.center + zone.band)}`);
    } else {
      // Scharfes Level
      const color = side === 'support' ? SUPPORT_COLOR : RESISTANCE_COLOR;
      const opacity = mapConfidenceToOpacity(zone.confidence);

      renderSolidLine(zone.center, color, opacity);
      renderFilledRect(                                          // dezenter Halo
        zone.center - zone.band,
        zone.center + zone.band,
        color,
        HALO_OPACITY
      );
      renderLabel(`${formatPrice(zone.center)} ${zone.sources}`);
    }
  }
}
```

---

## Backend-Änderungen

### SrZone.java (odin-brain)

Neues Feld/Enum:

```java
public enum ZoneType {
    LEVEL,           // Scharfes S/R-Level
    BAND             // Konsolidierungszone
}
```

`SrZone` bekommt `ZoneType zoneType()`. Default: `LEVEL`.
ConsolidationBands werden mit `BAND` erstellt.

### SrZoneDto-Mapping (odin-app)

```java
new SrZoneDto(
    zone.center(),
    zone.band(),
    zone.confidence(),
    zone.touchCount(),
    zone.sourceLabel(),
    zone.zoneType().name()   // "LEVEL" oder "BAND"
)
```

### SSE-Events (für Live-Pipeline)

Wenn die Live-Pipeline S/R-Updates streamt, enthält das SSE-Event ebenfalls
`zoneType`. Keine separate Event-Struktur nötig.

---

## Interaktion mit anderen Konzepten

| Konzept | Interaktion |
|---------|-------------|
| Konzept 1 (OR-Filter) | Gefilterte OR-Level sind immer `LEVEL` (nie `BAND`) |
| Konzept 2 (Konsolidierung) | ConsolidationBand → `zoneType=BAND` mit breitem `band` |
| Konzept 3 (Dynamic Scoring) | Quality-Berechnung identisch für LEVEL und BAND |
| Konzept 4 (Spatial Registry) | BAND-touchCount kommt aus Registry-Range-Query |
| Konzept 5 (Output Quality) | BAND-Confidence via Base+Boost, eigener Prior möglich |

---

## Risiken und Mitigationen

| Risiko | Mitigation |
|--------|------------|
| Breaking Change im API-Contract | `zoneType` ist additiv; bestehende Felder unverändert |
| Frontend muss BAND-Rendering implementieren | Graceful Degradation: unbekannter zoneType → als LEVEL rendern |
| Zu viele Bänder überlagern den Chart | Typisch 0–2 Bänder pro Session; OutputPolicy begrenzt |
| Band-Breite visuell dominant | Opacity 12–18% ist dezent; POC als Orientierungspunkt |
| LLM/Rules brauchen die Semantik | `CONSOLIDATION` Source + `BAND` Type reichen als Signal |

## Tests

### Backend
1. ConsolidationBand wird als SrZoneDto mit zoneType="BAND" serialisiert
2. Reguläres Level wird als SrZoneDto mit zoneType="LEVEL" serialisiert
3. API-Response enthält gemischte LEVEL- und BAND-Zonen
4. BAND-center ist POC, BAND-band ist halbe Breite

### Frontend
1. LEVEL-Zone: Linie + Halo gerendert
2. BAND-Zone: Gefülltes Rechteck + gestrichelte Ränder + POC
3. Unbekannter zoneType: Fallback auf LEVEL-Rendering
4. Konsolidierungsfarbe (Orange) unterscheidet sich von Support/Resistance
5. Halo-Opacity < Band-Opacity (Halo dezenter als Band)
