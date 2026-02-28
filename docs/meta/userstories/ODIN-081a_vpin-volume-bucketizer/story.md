# ODIN-081a: VPIN — Volume-Bucket-Segmentierung und Bulk Volume Classification

**Modul:** odin-brain
**Phase:** 1
**Abhaengigkeiten:** Keine
**Geschaetzter Umfang:** M

> **Hinweis:** Diese Story ist Teil der aufgeteilten XL-Story ODIN-081 (VPIN).
> Abhaengigkeitskette: **ODIN-081a** → ODIN-081b → ODIN-081c

---

## Kontext

VPIN (Volume-Synchronized Probability of Informed Trading) quantifiziert Orderflow-Toxizitaet durch Volume-Bucket-Analyse statt zeitbasierter Betrachtung. Diese Sub-Story implementiert die beiden Basiskomponenten des VPIN-Algorithmus: den `VolumeBucketizer`, der 1m-Bars in feste Volumenscheiben zerlegt, und den `BulkVolumeClassifier`, der jede Scheibe in Buy- und Sell-Volumen klassifiziert. Diese beiden Klassen bilden die Eingabe fuer die VPIN-Berechnung (ODIN-081b) und muessen inkrementell und zustandslos (abgesehen von internem Buffer) arbeiten. Akademische Grundlage: Easley, Lopez de Prado, O'Hara (2012).

## Scope

**In Scope:**

- Immutable Record `VolumeBucket` mit Feldern `bucketVolume`, `buyVolume`, `sellVolume`
- Klasse `VolumeBucketizer`: segmentiert 1m-Bars inkrementell in Volume-Buckets fester Groesse
- Klasse `BulkVolumeClassifier`: klassifiziert Bucket-Volumen in Buy/Sell-Anteil (Bar-Approximation)
- Unit-Tests fuer beide Klassen mit diversen Volume-Mustern und Edge Cases
- Package: `de.its.odin.brain.kpi`

**Out of Scope:**

- VPIN-Berechnung (Rolling-Average) — das ist ODIN-081b
- Integration in KpiEngine oder IndicatorResult — das ist ODIN-081b
- Gate-Integration, DataFlag, Persistence — das ist ODIN-081c
- Tick-Level-Daten (nur Bar-Approximation, keine echte BVC-Formel nach Easley et al. mit CDF)
- Konfigurationsparameter (werden in ODIN-081b als Teil der VpinProperties eingefuehrt)

## Akzeptanzkriterien

### VolumeBucket Record

- [ ] Immutable Record `VolumeBucket` im Package `de.its.odin.brain.kpi`
- [ ] Felder: `double bucketVolume`, `double buyVolume`, `double sellVolume`
- [ ] Berechnetes Attribut oder Hilfsmethode `sellVolume()` konsistent mit `buyVolume + sellVolume = bucketVolume`
- [ ] JavaDoc vollstaendig

### VolumeBucketizer

- [ ] Neue Klasse `VolumeBucketizer` im Package `de.its.odin.brain.kpi`
- [ ] Konstruktor nimmt `double targetBucketVolume` als Parameter
- [ ] Methode `List<VolumeBucket> addBar(Bar bar)` — verarbeitet eine 1m-Bar inkrementell; gibt alle in diesem Schritt abgeschlossenen Buckets zurueck (kann leer, eins oder mehrere sein)
- [ ] Methode `Optional<VolumeBucket> getPartialBucket()` — gibt den aktuell in Bildung befindlichen (unvollstaendigen) Bucket zurueck
- [ ] Methode `void reset()` — setzt internen Zustand zurueck (fuer Tageswechsel)
- [ ] Ein Bar-Volumen kann mehrere Buckets abschliessen (z.B. bei Volume-Spikes)
- [ ] Unvollstaendige Buckets werden NICHT verworfen — sie bleiben als Partial-Bucket erhalten
- [ ] Inkrementelle Verarbeitung: interner Zustand haelt akkumuliertes Volumen des laufenden Buckets
- [ ] Unit-Test: 10 Bars mit je 100 Volumen, Bucket-Size 250 → 4 abgeschlossene Buckets (je 250), Rest 0
- [ ] Unit-Test: 3 Bars mit Volumen [100, 400, 50], Bucket-Size 250 → Bucket 1 (100+150=250 aus Bar 2), Bucket 2 (250 aus Bar 2), Partial 50
- [ ] Unit-Test: Volume-Spike (eine Bar mit 1000 Volumen, Bucket-Size 300) → 3 Buckets (300, 300, 300), Partial 100
- [ ] Unit-Test: reset() loescht alle internen Zustaende

### BulkVolumeClassifier

- [ ] Neue Klasse `BulkVolumeClassifier` im Package `de.its.odin.brain.kpi`
- [ ] Methode `double classifyBuyRatio(Bar bar)` — berechnet Buy-Ratio fuer eine Bar
- [ ] BVC-Approximation (Bar-basiert statt Tick-basiert): `buyRatio = 0.5 + 0.5 * (close - open) / (high - low)` wenn `high != low`
- [ ] Edge Case `high == low` (Doji ohne Range): `buyRatio = 0.5`
- [ ] Edge Case `close < open` und `high == low` (Flat-Doji): `buyRatio = 0.5`
- [ ] `buyRatio` liegt immer im Bereich [0.0, 1.0] — Clamp falls numerische Ausreisser auftreten
- [ ] Methode `VolumeBucket classify(double totalVolume, double buyRatio)` — erstellt VolumeBucket aus Gesamtvolumen und Buy-Ratio
- [ ] Unit-Test: Bullish Bar (close = 105, open = 100, high = 106, low = 99) → buyRatio > 0.7
- [ ] Unit-Test: Bearish Bar (close = 95, open = 100, high = 101, low = 94) → buyRatio < 0.3
- [ ] Unit-Test: Doji Bar (close = 100, open = 100, high = 102, low = 98) → buyRatio = 0.5
- [ ] Unit-Test: high == low (Flat Bar) → buyRatio = 0.5 (kein Division-by-Zero)
- [ ] Unit-Test: Clamp-Pruefung — Ergebnis immer in [0.0, 1.0]

## Technische Details

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `VolumeBucket` | `de.its.odin.brain.kpi` | Immutable Record: bucketVolume, buyVolume, sellVolume |
| `VolumeBucketizer` | `de.its.odin.brain.kpi` | Segmentiert 1m-Bars inkrementell in Volume-Buckets |
| `BulkVolumeClassifier` | `de.its.odin.brain.kpi` | BVC: Buy/Sell-Klassifikation pro Bucket via Bar-Approximation |

### Bestehende Klassen (werden NICHT veraendert)

Keine — diese Sub-Story ist rein additiv und beruehrt keine bestehenden Klassen.

### Bar-DTO

Die `Bar`-Klasse liegt in `odin-api` unter `de.its.odin.api.dto.Bar` und stellt `open()`, `close()`, `high()`, `low()`, `volume()`, `closeTime()` bereit.

### BVC-Formel (Bar-Approximation)

```
buyRatio = clamp(0.5 + 0.5 * (close - open) / (high - low), 0.0, 1.0)
```

- Wenn `high == low`: `buyRatio = 0.5`
- Vollstaendig bullische Bar (close == high, open == low): buyRatio = 1.0
- Vollstaendig bearische Bar (close == low, open == high): buyRatio = 0.0
- Diese Approximation erklaert 85-90% der Varianz des Tick-basierten VPIN (Forschungsergebnis)

### Inkrementelle Verarbeitung des VolumeBucketizer

```
State: partialVolume, partialBuyVolume, partialSellVolume

addBar(bar):
  buyRatio = classifier.classifyBuyRatio(bar)
  barBuyVol = bar.volume() * buyRatio
  barSellVol = bar.volume() * (1 - buyRatio)
  remaining = bar.volume()
  completedBuckets = []
  while (partialVolume + remaining >= targetBucketVolume):
    needed = targetBucketVolume - partialVolume
    fraction = needed / remaining
    completedBuckets.add(new VolumeBucket(...))
    partialVolume = 0
    remaining -= needed
    partialBuyVolume = 0; partialSellVolume = 0
  // rest goes into partial bucket
  partialVolume += remaining
  partialBuyVolume += remaining * buyRatio
  partialSellVolume += remaining * (1 - buyRatio)
  return completedBuckets
```

### Konstanten

```java
private static final double BUY_RATIO_NEUTRAL = 0.5;
private static final double BUY_RATIO_MIN = 0.0;
private static final double BUY_RATIO_MAX = 1.0;
```

## Konzept-Referenzen

- `docs/meta/userstories/ODIN-081_vpin/story.md` — Uebergeordnete Story, Kontext und akademische Grundlage
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Engine-Architektur, Indikator-Berechnung, Bar-Datenfluss
- `docs/backend/architecture/02-realtime-pipeline.md` — 1m-Bar-Verarbeitung, DataPipelineService, Bar-DTO
- Akademische Referenz: Easley, Lopez de Prado, O'Hara (2012): "Flow Toxicity and Liquidity in a High Frequency World"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln (odin-brain darf nur odin-api nutzen)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln: Records fuer DTOs, keine Magic Numbers, kein var, JavaDoc-Pflicht

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (`VolumeBucket` als Record)
- [ ] ENUM statt String fuer endliche Mengen (nicht anwendbar in dieser Story)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: nicht anwendbar (keine Konfiguration in dieser Story)
- [ ] Port-Abstraktion: nicht anwendbar (keine Port-Interfaces benoetigt)

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `VolumeBucketizer`: normale Faelle, Volume-Spike, Partial-Bucket, reset()
- [ ] Unit-Tests fuer `BulkVolumeClassifier`: bullish, bearish, doji, flat bar (high==low), clamp
- [ ] Testklassen-Namenskonvention: `VolumeBucketizerTest`, `BulkVolumeClassifierTest` (Surefire)
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: `VolumeBucketizer` + `BulkVolumeClassifier` zusammen — kompletter Datenpfad von Bars zu klassifizierten Buckets
- [ ] Integrationstest: Sequenz von 50+ Bars → korrekte Bucket-Anzahl und Buy/Sell-Summen
- [ ] Testklassen-Namenskonvention: `VpinBucketizerIntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank (falls zutreffend)
- [ ] Nicht zutreffend — diese Story hat keinen Datenbankzugriff

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: Klassen, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Division-by-Zero, negative Volumina, Volume-Spikes ueber viele Buckets, Floating-Point-Praezision bei Bucket-Aufteilung
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review — Bugs, Null-Safety, Floating-Point-Akkumulation, Division-by-Zero-Schutz
- [ ] Dimension 2: Konzepttreue-Review — BVC-Formel korrekt implementiert, Bucket-Logik entspricht Konzept
- [ ] Dimension 3: Praxis-Review — Verhalten bei Pre-Market (niedriges Volumen), bei Volume-Spikes, Floating-Point-Drift bei langer Laufzeit
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis wird waehrend der Arbeit live aktualisiert
- [ ] Pflichtabschnitte: Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### Floating-Point-Praezision

Bei der inkrementellen Bucket-Befuellung akkumuliert sich Floating-Point-Fehler ueber viele Bars. Die Implementierung sollte robust damit umgehen — ein kleiner Toleranzbereich bei Vergleichen mit `targetBucketVolume` ist akzeptabel. Kein Epsilon-Einfuehren ohne konkreten Testfall.

### Inkrementelle Verarbeitung vs. Batch

`VolumeBucketizer` muss INKREMENTELL arbeiten — eine Bar nach der anderen, nicht Batch. Der Zustand (Partial-Bucket) muss zwischen `addBar()`-Aufrufen erhalten bleiben. `reset()` fuer Tageswechsel.

### Clamp-Logik in BulkVolumeClassifier

Die BVC-Formel kann bei sehr kleinen `(high - low)` Intervallen theoretisch Werte ausserhalb [0, 1] produzieren. Ein expliziter `Math.max(0.0, Math.min(1.0, value))`-Clamp ist Pflicht.

### Volumen-Split bei Bucket-Grenze

Wenn eine Bar ein Bucket abschliesst und noch Volume uebrig hat: Das restliche Volume geht proportional (mit der gleichen buyRatio der Bar) in den naechsten Bucket. Die buyRatio wird NICHT neu berechnet — eine Bar hat eine Richtung.

### Keine Spring-Beans

`VolumeBucketizer` und `BulkVolumeClassifier` sind POJOs ohne Spring-Annotation. Sie werden von `VpinCalculator` (ODIN-081b) instanziiert und gehalten.
