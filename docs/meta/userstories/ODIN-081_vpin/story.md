# ODIN-081: VPIN (Volume-Synchronized Probability of Informed Trading)

**Modul:** odin-data, odin-brain, odin-api, odin-core
**Phase:** 1
**Abhaengigkeiten:** Keine
**Geschaetzter Umfang:** XL

---

## Kontext

VPIN (Volume-Synchronized Probability of Informed Trading) quantifiziert Orderflow-Toxizitaet: Wie wahrscheinlich ist es, dass informierte institutionelle Akteure den Orderflow dominieren? Im Gegensatz zu zeitbasierten Indikatoren verwendet VPIN Volume-Buckets — feste Volumenscheiben statt Zeitintervalle — was die Analyse von der Tageszeit entkoppelt und Phasen institutioneller Aktivitaet zuverlaessiger erkennt. Ein steigender VPIN signalisiert toxischen Orderflow und sollte zu konserativerem Trading fuehren (keine neuen Entries, Stops enger ziehen).

## Scope

**In Scope:**

- Volume-Bucket-Segmentierung: Tick- oder 1m-Bar-Daten in feste Volumeneinheiten aufteilen
- Bulk Volume Classification (BVC): Volumen jedes Buckets in Buy- und Sell-Anteil klassifizieren
- VPIN-Berechnung: Rolling-Average der |Buy - Sell| / BucketVolume ueber N Buckets
- Neuer KPI `vpin` in `IndicatorResult`
- Integration als Gate-Modifier: VPIN ueber Schwelle → konservativere Entry-Schwellen
- Integration als Stop-Modifier: VPIN ueber Schwelle → engerer Trailing-Stop
- Konfigurationsparameter (Bucket-Size, Window, Thresholds)

**Out of Scope:**

- Tick-Level-Daten von IB TWS API (vorerst Bar-basierte Approximation mit 1m-Bars)
- VPIN als eigenstaendiges Gate in der Gate-Cascade (wird als Modifier implementiert, nicht als Hard-Gate)
- Echtzeit-VPIN-Visualisierung im Frontend (separate Story)
- Historisches VPIN-Monitoring und Alerting

## Akzeptanzkriterien

### Volume-Bucket-Segmentierung

- [ ] Neue Klasse `VolumeBucketizer` segmentiert 1m-Bars in Volume-Buckets fester Groesse
- [ ] Bucket-Groesse konfigurierbar (Default: Average Daily Volume / 50)
- [ ] Unvollstaendige Buckets am Ende werden mitgenommen (nicht verworfen)
- [ ] Unit-Test: 10 Bars mit je 100 Volumen, Bucket-Size 250 → 4 Buckets (250, 250, 250, 250)
- [ ] Unit-Test: Bars mit stark variierendem Volumen → korrekte Bucket-Zuordnung

### Bulk Volume Classification (BVC)

- [ ] Neue Klasse `BulkVolumeClassifier` klassifiziert Bucket-Volumen in Buy/Sell-Anteil
- [ ] BVC-Formel nach Easley/Lopez de Prado/O'Hara (2012): `buyVolume = V * CDF(Z(deltaPrice / sigma))`
- [ ] Vereinfachte Implementierung: `buyRatio = 0.5 + 0.5 * (close - open) / (high - low)` wenn Bar-Daten statt Tick-Daten
- [ ] Unit-Test: Bullish Bar (close >> open) → buyRatio > 0.7
- [ ] Unit-Test: Bearish Bar (close << open) → buyRatio < 0.3
- [ ] Unit-Test: Doji (close ≈ open) → buyRatio ≈ 0.5

### VPIN-Berechnung

- [ ] Neue Klasse `VpinCalculator` berechnet Rolling-VPIN ueber N Buckets
- [ ] VPIN-Formel: `VPIN = (1/N) * Σ|BuyVolume_i - SellVolume_i| / BucketVolume`
- [ ] Window-Groesse konfigurierbar (Default: 50 Buckets)
- [ ] VPIN-Wert liegt im Bereich [0, 1]: 0 = balanced Flow, 1 = vollstaendig einseitig
- [ ] Unit-Test: Balanced Buckets (50/50 Buy/Sell) → VPIN ≈ 0.0
- [ ] Unit-Test: Vollstaendig einseitige Buckets (100% Buy) → VPIN ≈ 1.0
- [ ] Unit-Test: Gemischte Buckets → VPIN im Bereich [0.2, 0.5]

### Integration in ODIN

- [ ] `IndicatorResult` erhaelt neues Feld `double vpin` (NaN bis genuegend Buckets vorliegen)
- [ ] `KpiEngine` aktualisiert VPIN bei jedem 1m-Bar
- [ ] VPIN ueber konfigurierbarer Schwelle (Default: 0.7) → `DataFlag.VPIN_ELEVATED` wird gesetzt
- [ ] `GateCascadeEvaluator` beruecksichtigt `VPIN_ELEVATED`: bei aktivem Flag werden Gate-Schwellen auf elevated-Modus geschaltet (analoge Logik zu `regimeConfidence < 0.7`)
- [ ] Neues `DataFlag.VPIN_ELEVATED` in `de.its.odin.api.model.DataFlag`
- [ ] Integrationstest: Kompletter Datenpfad von 1m-Bar → Bucketizer → BVC → VPIN → IndicatorResult → Gate-Modifier

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `IndicatorResult` | `odin-api/.../dto/IndicatorResult.java` | Neues Feld `double vpin` |
| `KpiEngine` | `odin-brain/.../kpi/KpiEngine.java` | VpinCalculator als Feld, Update bei jedem 1m-Bar |
| `DataFlag` | `odin-api/.../model/DataFlag.java` | Neuer Wert `VPIN_ELEVATED` |
| `GateCascadeEvaluator` | `odin-brain/.../quant/GateCascadeEvaluator.java` | VPIN_ELEVATED → elevated Schwellen |
| `BrainProperties` | `odin-brain/.../config/BrainProperties.java` | Neues `VpinProperties` in `KpiProperties` |
| `IndicatorSnapshotEntity` | `odin-brain/.../persistence/IndicatorSnapshotEntity.java` | Neues Feld `vpin` |

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `VolumeBucketizer` | `de.its.odin.brain.kpi` | Segmentiert 1m-Bars in Volume-Buckets |
| `BulkVolumeClassifier` | `de.its.odin.brain.kpi` | BVC: Buy/Sell-Klassifikation pro Bucket |
| `VpinCalculator` | `de.its.odin.brain.kpi` | Rolling-VPIN-Berechnung ueber N Buckets |
| `VolumeBucket` | `de.its.odin.brain.kpi` | Immutable Record: bucketVolume, buyVolume, sellVolume |

### Konfiguration

```properties
# VPIN configuration
odin.brain.kpi.vpin.enabled=true
odin.brain.kpi.vpin.bucket-volume-divisor=50
odin.brain.kpi.vpin.window-size=50
odin.brain.kpi.vpin.elevated-threshold=0.70
odin.brain.kpi.vpin.critical-threshold=0.85
```

### Datenfluss

```
1m-Bar → VolumeBucketizer → VolumeBucket[] → BulkVolumeClassifier → BuyVol/SellVol
  → VpinCalculator → VPIN (0-1) → IndicatorResult.vpin → GateCascadeEvaluator (Flag-Check)
```

## Konzept-Referenzen

- `theme-backlog.md` Thema 20: "VPIN (Volume-Synchronized Probability of Informed Trading)" — IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Engine, Indikator-Berechnung
- `docs/backend/architecture/02-realtime-pipeline.md` — 1m-Bar-Verarbeitung, DataPipelineService
- Akademische Referenz: Easley, Lopez de Prado, O'Hara (2012): "Flow Toxicity and Liquidity in a High Frequency World"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Cross-Modul-Abhaengigkeiten
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln (Records fuer DTOs, keine Magic Numbers, @ConfigurationProperties)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api,odin-brain,odin-data`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (VolumeBucket)
- [ ] ENUM statt String (VPIN_ELEVATED in DataFlag)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.kpi.vpin.*`

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `VolumeBucketizer` (verschiedene Volume-Muster)
- [ ] Unit-Tests fuer `BulkVolumeClassifier` (bullish, bearish, neutral)
- [ ] Unit-Tests fuer `VpinCalculator` (balanced, einseitig, gemischt, warmup)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Vollstaendiger VPIN-Datenpfad von Bars bis IndicatorResult
- [ ] Integrationstest: GateCascadeEvaluator mit VPIN_ELEVATED Flag
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen, Akzeptanzkriterien, Tests
- [ ] Edge Cases: Sehr kleine Buckets, Volume-Spikes, Pre-Market vs. RTH
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Numerik-Praezision, Division-by-Zero bei leeren Buckets)
- [ ] Dimension 2: Konzepttreue-Review (Vergleich mit Easley et al., Thema 20)
- [ ] Dimension 3: Praxis-Review (VPIN bei Small-Cap vs. Large-Cap, Aussagekraft bei 1m-Bars statt Ticks)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### XL-Story: Empfehlung fuer Sub-Story-Aufteilung

Diese Story ist XL und koennte in 3 Sub-Stories aufgeteilt werden:

1. **ODIN-081a: Volume-Bucket-Segmentierung + BVC** (M) — `VolumeBucketizer`, `BulkVolumeClassifier`, `VolumeBucket` Record, Unit-Tests
2. **ODIN-081b: VPIN-Berechnung + KPI-Integration** (M) — `VpinCalculator`, Integration in `KpiEngine`, `IndicatorResult.vpin`, Unit-Tests + Integrationstests
3. **ODIN-081c: Gate-Integration + Persistence** (S) — `DataFlag.VPIN_ELEVATED`, `GateCascadeEvaluator`-Erweiterung, `IndicatorSnapshotEntity`-Erweiterung, Flyway-Migration

Die Aufteilung haengt von der Kapazitaet und der gewuenschten Parallelisierbarkeit ab. Sub-Stories a und b koennen teilweise parallel arbeiten (BVC wird von VPIN benoetigt).

### Technische Hinweise

- **Bar-basierte Approximation statt Tick-Daten:** Die BVC-Formel von Easley et al. ist fuer Tick-Daten konzipiert. Da ODIN ueber IB TWS API nur Bar-Daten (1m) erhaelt, verwenden wir die vereinfachte Approximation `buyRatio = 0.5 + 0.5 * (close - open) / (high - low)`. Diese ist weniger praezise als die Original-Formel, aber fuer Intraday-Zwecke ausreichend. Die Literatur zeigt, dass Bar-basiertes VPIN 85-90% der Varianz des Tick-basierten VPIN erklaert.
- **Bucket-Size-Kalibrierung:** Die Bucket-Size sollte so gewaehlt werden, dass pro Handelstag 40-60 Buckets entstehen (Research-Empfehlung). Bei einem Average Daily Volume von 5M Shares waere die Bucket-Size ca. 100K Shares. Der Divisor (Default 50) berechnet sich dynamisch aus dem geschaetzten Tagesvolumen.
- **Warmup:** VPIN benoetigt mindestens `window-size` Buckets (Default: 50) bevor ein valider Wert geliefert wird. Vorher: `Double.NaN`. Bei normalem Tagesvolumen sind das ca. 60-90 Minuten nach RTH-Open.
- **Flyway-Migration:** Neues Feld `vpin` in der `indicator_snapshot`-Tabelle. Nullable, da bestehende Datensaetze keinen VPIN-Wert haben.
- **Performance:** `VolumeBucketizer` und `VpinCalculator` muessen inkrementell arbeiten (pro neuem 1m-Bar, nicht komplett neu berechnen). State wird pro Pipeline-Instanz gehalten.
