# ODIN-081b: VPIN — Berechnung und KPI-Engine-Integration

**Modul:** odin-brain, odin-api
**Phase:** 1
**Abhaengigkeiten:** ODIN-081a (VolumeBucketizer, BulkVolumeClassifier, VolumeBucket muessen vorliegen)
**Geschaetzter Umfang:** M

> **Hinweis:** Diese Story ist Teil der aufgeteilten XL-Story ODIN-081 (VPIN).
> Abhaengigkeitskette: ODIN-081a → **ODIN-081b** → ODIN-081c

---

## Kontext

Nachdem ODIN-081a den Datenstrom in klassifizierte Volume-Buckets zerlegt hat, berechnet diese Sub-Story den eigentlichen VPIN-Wert als Rolling-Average ueber N Buckets. Der Wert wird in `IndicatorResult` integriert und steht damit dem gesamten Decision-Loop zur Verfuegung. Die Integration in `KpiEngine` muss inkrementell sein — bei jedem neuen 1m-Bar wird der VPIN-Zustand aktualisiert, nicht neu berechnet. Warmup-Periode: mindestens `window-size` (Default 50) Buckets, bis dahin `Double.NaN`.

## Scope

**In Scope:**

- Neue Klasse `VpinCalculator`: Rolling-VPIN-Berechnung ueber N Buckets als Circular-Buffer
- Erweiterung von `IndicatorResult` (odin-api): neues Feld `double vpin`
- Erweiterung von `KpiEngine` (odin-brain): `VpinCalculator` als Feld, VPIN-Update bei jedem 1m-Bar
- Neue Konfigurationsgruppe `VpinProperties` in `BrainProperties.KpiProperties`
- Erweiterung der `isWarmupComplete`-Logik: VPIN-Warmup NICHT als Blocker (VPIN ist optional)
- Unit-Tests fuer `VpinCalculator`
- Integrationstests: Datenpfad Bar → Bucketizer → BVC → VpinCalculator → IndicatorResult

**Out of Scope:**

- Gate-Integration (`GateCascadeEvaluator`, `DataFlag.VPIN_ELEVATED`) — das ist ODIN-081c
- Persistence (`IndicatorSnapshotEntity`, Flyway-Migration) — das ist ODIN-081c
- Frontend-Anzeige von VPIN

## Akzeptanzkriterien

### VpinCalculator

- [ ] Neue Klasse `VpinCalculator` im Package `de.its.odin.brain.kpi`
- [ ] Konstruktor: `VpinCalculator(int windowSize, double targetBucketVolume)` — instanziiert intern `VolumeBucketizer` und `BulkVolumeClassifier`
- [ ] Methode `double onBar(Bar bar)` — verarbeitet eine 1m-Bar; gibt aktuellen VPIN-Wert oder `Double.NaN` zurueck
- [ ] VPIN-Formel: `VPIN = (1/N) * Σ |buyVolume_i - sellVolume_i| / bucketVolume_i` ueber die letzten N Buckets
- [ ] Circular-Buffer fuer die letzten N Buckets (kein unbegrenztes Wachstum)
- [ ] VPIN-Wert liegt immer im Bereich [0.0, 1.0]
- [ ] Gibt `Double.NaN` zurueck solange weniger als `windowSize` Buckets gesammelt wurden (Warmup)
- [ ] Methode `void reset()` — setzt internen Zustand zurueck (inkl. Bucketizer und Buffer)
- [ ] Unit-Test: Balanced Buckets (je 50% buy/sell) → VPIN = 0.0
- [ ] Unit-Test: Vollstaendig einseitige Buckets (100% buy) → VPIN = 1.0
- [ ] Unit-Test: Warmup-Periode — VPIN = NaN solange < windowSize Buckets
- [ ] Unit-Test: Gemischte Buckets → VPIN in [0.2, 0.5]
- [ ] Unit-Test: Circular-Buffer-Verhalten — aelteste Buckets fallen heraus wenn Buffer voll

### IndicatorResult-Erweiterung (odin-api)

- [ ] `IndicatorResult` Record erhaelt neues Feld `double vpin` an letzter Position
- [ ] JavaDoc-Kommentar fuer `vpin`: "VPIN (Volume-Synchronized Probability of Informed Trading), range [0,1]; NaN during warmup or if VPIN is disabled"
- [ ] Alle bestehenden `IndicatorResult`-Konstruktoraufrufe in der Codebasis werden aktualisiert (Compiler-Fehler vermeiden)
- [ ] `isWarmupComplete` in `KpiEngine` wird NICHT von VPIN abhaengig gemacht — VPIN ist ein optionaler Indikator

### BrainProperties-Erweiterung

- [ ] Neue Nested-Record `VpinProperties` in `BrainProperties.KpiProperties`
- [ ] Felder: `boolean enabled`, `int bucketVolumeDivisor`, `int windowSize`, `double elevatedThreshold`, `double criticalThreshold`
- [ ] `@Validated`-Annotationen: `@Min(1)` fuer Divisor und Window, `@DecimalMin("0.0")` fuer Thresholds
- [ ] `KpiProperties` erhaelt das neue Feld `@NotNull @Valid VpinProperties vpin`
- [ ] Default-Werte in `application.properties` (odin-app):
  ```properties
  odin.brain.kpi.vpin.enabled=true
  odin.brain.kpi.vpin.bucket-volume-divisor=50
  odin.brain.kpi.vpin.window-size=50
  odin.brain.kpi.vpin.elevated-threshold=0.70
  odin.brain.kpi.vpin.critical-threshold=0.85
  ```

### KpiEngine-Erweiterung

- [ ] `KpiEngine` haelt `VpinCalculator` als Instanzfeld (oder `null` wenn `vpin.enabled=false`)
- [ ] `VpinCalculator` wird im Konstruktor von `KpiEngine` erstellt (wenn enabled)
- [ ] `onSnapshot()` ruft `vpinCalculator.onBar(neuesteBar)` auf — nur wenn mindestens eine neue 1m-Bar vorliegt
- [ ] `reset()` in `KpiEngine` ruft `vpinCalculator.reset()` auf
- [ ] Wenn `vpin.enabled=false`: `vpin`-Feld in `IndicatorResult` bleibt `Double.NaN`
- [ ] `targetBucketVolume` wird zur Laufzeit aus dem `averageDailyVolume`-Schaetzer berechnet (Fallback: konfigurierter Divisor als Fallback-Bucket-Size, z.B. 100_000 / divisor wenn kein ADV bekannt)

### Integrationstests

- [ ] Integrationstest: Vollstaendiger VPIN-Datenpfad — 100 Bars simuliert → `VpinCalculator` → `IndicatorResult.vpin` nicht NaN nach Warmup
- [ ] Integrationstest: `KpiEngine` mit aktiviertem VPIN — nach 50+ vollstaendigen Buckets liefert `onSnapshot()` einen validen VPIN-Wert

## Technische Details

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `VpinCalculator` | `de.its.odin.brain.kpi` | Rolling-VPIN-Berechnung mit Circular-Buffer, haelt intern Bucketizer + BVC |

### Geaenderte Klassen

| Klasse | Datei | Aenderung |
|--------|-------|-----------|
| `IndicatorResult` | `odin-api/.../dto/IndicatorResult.java` | Neues Feld `double vpin` (letzte Position); alle Konstruktoraufrufe anpassen |
| `KpiEngine` | `odin-brain/.../kpi/KpiEngine.java` | VpinCalculator als Feld; `onSnapshot()` und `reset()` erweitern |
| `BrainProperties` | `odin-brain/.../config/BrainProperties.java` | Neues `VpinProperties` in `KpiProperties` |

### IndicatorResult-Record nach Erweiterung (Auszug)

```java
public record IndicatorResult(
    // ... bestehende Felder ...
    boolean warmupComplete,
    Set<DataFlag> flags,
    Subregime subregime,
    double vpin   // NEU: NaN waehrend Warmup oder wenn disabled
) { ... }
```

### Circular-Buffer-Implementierung

Ein `ArrayDeque<VolumeBucket>` der Groesse `windowSize` genuegt. Bei Vollstand wird der aelteste Bucket mit `pollFirst()` entfernt vor dem `addLast()` des neuen Buckets.

### VPIN-Formel

```java
double vpin = bucketBuffer.stream()
    .mapToDouble(b -> Math.abs(b.buyVolume() - b.sellVolume()) / b.bucketVolume())
    .average()
    .orElse(Double.NaN);
```

### Bucket-Volume-Kalibrierung

Da der `averageDailyVolume` zur Story-Implementierungszeit nicht aus dem Datenstrom verfuegbar sein wird (erfordert historische Daten), wird folgendes Fallback-Schema verwendet:
- Wenn ADV nicht bekannt: feste Bucket-Size = `100_000 / bucketVolumeDivisor` (d.h. 2_000 bei Default-Divisor 50)
- Die ADV-basierte dynamische Kalibrierung ist eine spaetere Erweiterung

### Aufruf in KpiEngine.onSnapshot()

```java
// VPIN: update on each new 1m bar
double vpin = Double.NaN;
if (vpinCalculator != null) {
    List<Bar> bars1m = snapshot.oneMinBars();
    if (!bars1m.isEmpty()) {
        Bar latestBar = bars1m.stream()
            .max(Comparator.comparing(Bar::closeTime))
            .orElseThrow();
        if (latestBar.closeTime().isAfter(lastAdded1mBarTime before vpin update)) {
            vpin = vpinCalculator.onBar(latestBar);
        }
    }
}
```

## Konzept-Referenzen

- `docs/meta/userstories/ODIN-081_vpin/story.md` — Uebergeordnete Story; Konfiguration, Warmup-Details
- `docs/meta/userstories/ODIN-081a_vpin-volume-bucketizer/story.md` — Abhaengige Story (Bucketizer + BVC)
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Engine-Architektur, Instanziierungsmodell (Pro-Pipeline-POJO)
- `docs/backend/architecture/02-realtime-pipeline.md` — 1m-Bar-Verarbeitung, `MarketSnapshot`-Struktur
- Akademische Referenz: Easley, Lopez de Prado, O'Hara (2012)

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Abhaengigkeitsregeln: odin-api hat keine Abhaengigkeiten; odin-brain darf odin-api nutzen
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln: keine Magic Numbers, kein var, Records fuer DTOs, @ConfigurationProperties als Record + @Validated

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api,odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (Window-Size-Defaults, Bucket-Fallback-Volume)
- [ ] Records fuer DTOs (VpinProperties als nested Record in BrainProperties)
- [ ] ENUM statt String fuer endliche Mengen (nicht anwendbar)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.kpi.vpin.*`
- [ ] Port-Abstraktion: nicht anwendbar (interne KPI-Komponenten)

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `VpinCalculator`: balanced, einseitig, warmup (NaN), circular-buffer-rollover
- [ ] Unit-Tests fuer `KpiEngine` mit VPIN-enabled (VPIN-Feld in IndicatorResult nach Warmup valide)
- [ ] Testklassen-Namenskonvention: `VpinCalculatorTest`, `KpiEngineTest` erweitert (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Vollstaendiger VPIN-Datenpfad — 100 Bars simuliert → IndicatorResult.vpin valide nach Warmup
- [ ] Integrationstest: KpiEngine mit VPIN disabled → IndicatorResult.vpin = NaN
- [ ] Testklassen-Namenskonvention: `VpinCalculatorIntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank (falls zutreffend)
- [ ] Nicht zutreffend — diese Story hat keinen Datenbankzugriff (Persistence kommt in ODIN-081c)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: Klassen, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Warmup-Grenzfall (genau N Buckets), Floating-Point-Akkumulation im Circular-Buffer, Division-by-Zero bei Bucket mit Volume=0, Verhalten wenn alle Bars selbes Volumen
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review — Circular-Buffer korrekt implementiert, kein Memory-Leak, Null-Safety bei Bucketizer
- [ ] Dimension 2: Konzepttreue-Review — VPIN-Formel korrekt, Warmup-Semantik stimmt, KpiEngine-Integration entspricht Architektur
- [ ] Dimension 3: Praxis-Review — Bucket-Size-Kalibrierung realistisch? VPIN bei duennem Markt (Pre-Market)? Performance bei 390 Bars/Tag?
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis wird waehrend der Arbeit live aktualisiert
- [ ] Pflichtabschnitte: Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### IndicatorResult ist ein Record — alle Aufrufstellen updaten

`IndicatorResult` ist ein Record, daher muss bei Ergaenzung des Feldes `vpin` JEDE Konstruktoraufruf-Stelle im Backend angepasst werden. Mit `mvn compile -pl odin-brain` und anderen Modulen sofort pruefen — Compiler-Fehler zeigen alle Stellen.

### VPIN ist KEIN Warmup-Blocker

`isWarmupComplete()` in `KpiEngine` haengt NICHT von VPIN ab. VPIN benoetigt ~60-90 Minuten fuer valide Werte, waehrend das Trading-System typischerweise nach 20-30 Minuten operiert. VPIN ist ein optionaler Zusatz-Indikator, kein Pflichtwert fuer den Entry-Loop.

### Pro-Pipeline-Instanziierung

`VpinCalculator` ist kein Spring-Bean. Er wird in `KpiEngine` als Feld gehalten und im Konstruktor instanziiert. State ist pro Pipeline-Instanz. `PipelineFactory` erstellt `KpiEngine` (der dann `VpinCalculator` erstellt) — keine Aenderung an `PipelineFactory` noetig.

### Einmaliger Bar-Verbrauch

`onBar()` in `VpinCalculator` darf nicht zweimal mit derselben Bar aufgerufen werden. Die `KpiEngine` prueft bereits mit `lastAdded1mBarTime`, ob eine Bar neu ist. Diesen Mechanismus nutzen oder analog implementieren.
