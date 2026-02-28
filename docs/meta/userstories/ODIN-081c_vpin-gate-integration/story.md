# ODIN-081c: VPIN — Gate-Integration, DataFlag und Persistence

**Modul:** odin-api, odin-brain, odin-persistence
**Phase:** 1
**Abhaengigkeiten:** ODIN-081b (VpinCalculator integriert, IndicatorResult.vpin vorhanden)
**Geschaetzter Umfang:** S

> **Hinweis:** Diese Story ist Teil der aufgeteilten XL-Story ODIN-081 (VPIN).
> Abhaengigkeitskette: ODIN-081a → ODIN-081b → **ODIN-081c**

---

## Kontext

Nachdem VPIN in `IndicatorResult` verfuegbar ist (ODIN-081b), wird der Wert in dieser Sub-Story in den Decision-Loop integriert: Ein neuer `DataFlag.VPIN_ELEVATED` wird gesetzt wenn VPIN ueber der konfigurierten Schwelle liegt, und der `GateCascadeEvaluator` reagiert darauf mit strengeren Gate-Schwellen (analoges Muster zur bestehenden `elevated`-Logik fuer Regime-Confidence). Zusaetzlich wird VPIN in der `IndicatorSnapshotEntity` persistiert, damit historische VPIN-Werte fuer Chart-Overlays und Backtests verfuegbar sind.

## Scope

**In Scope:**

- Neuer Enum-Wert `VPIN_ELEVATED` in `DataFlag` (odin-api)
- VPIN-Flag-Setzung: `KpiEngine` oder ein neuer `VpinFlagEvaluator` setzt `VPIN_ELEVATED` wenn `vpin >= elevated-threshold`
- `GateCascadeEvaluator`-Erweiterung: wenn `VPIN_ELEVATED` im Flags-Set → `elevated = true` (zusaetzlich zur bestehenden Regime-Confidence-Logik)
- `IndicatorSnapshotEntity` (odin-brain): neues Feld `vpin` (nullable `Double`)
- Flyway-Migration V030: `ALTER TABLE odin.indicator_snapshot ADD COLUMN vpin DOUBLE PRECISION NULL`
- Anpassung der `IndicatorSnapshotEntity`-Konstruktor- und Getter-Methoden
- Unit-Tests fuer VPIN-Flag-Setzungslogik
- Integrationstest: VPIN_ELEVATED Flag → GateCascadeEvaluator schaltet auf elevated-Modus

**Out of Scope:**

- Frontend-Visualisierung von VPIN (separate Story)
- Kritischer-VPIN-Threshold (`critical-threshold=0.85`) — ggf. fuer spaetere Erweiterung reserviert, aber nicht implementiert
- Historisches VPIN-Monitoring und Alerting

## Akzeptanzkriterien

### DataFlag-Erweiterung (odin-api)

- [ ] `DataFlag` Enum erhaelt neuen Wert `VPIN_ELEVATED`
- [ ] JavaDoc-Kommentar: "VPIN exceeds the elevated threshold, indicating potentially toxic orderflow. Gate thresholds are tightened."
- [ ] Alle bestehenden Switch-Expressions und Pattern-Matches auf `DataFlag` pruefen (Compiler-Fehler zeigen fehlende Cases)

### VPIN-Flag-Setzung

- [ ] Nach VPIN-Berechnung in `KpiEngine.onSnapshot()`: wenn `!Double.isNaN(vpin) && vpin >= vpinProperties.elevatedThreshold()` → Flag `VPIN_ELEVATED` wird dem `flags`-Set des `IndicatorResult` hinzugefuegt
- [ ] Da `IndicatorResult` immutable ist, muss das Flag VOR der Konstruktion des Records gesetzt werden (im `onSnapshot()`-Aufruf)
- [ ] Das Flags-Set kommt urspruenglich aus `snapshot.flags()` (passthrough aus der Datenpipeline) — eine defensive Kopie als veraenderliches Set erstellen, Flag hinzufuegen, dann als `Set.copyOf()` an den Record uebergeben
- [ ] Unit-Test: `vpin = 0.75`, `elevatedThreshold = 0.70` → `VPIN_ELEVATED` im Flags-Set
- [ ] Unit-Test: `vpin = 0.65`, `elevatedThreshold = 0.70` → kein `VPIN_ELEVATED` im Flags-Set
- [ ] Unit-Test: `vpin = NaN` → kein `VPIN_ELEVATED` im Flags-Set (kein Flag waehrend Warmup)

### GateCascadeEvaluator-Erweiterung

- [ ] Der Aufrufer von `GateCascadeEvaluator.evaluate()` (vermutlich `DecisionArbiter` oder `RulesEngine`) leitet bereits `elevated` ab — der Aufrufer muss jetzt AUCH `IndicatorResult.flags().contains(DataFlag.VPIN_ELEVATED)` pruefen
- [ ] Wenn entweder Regime-Confidence im Bereich [0.5, 0.7] ODER `VPIN_ELEVATED` gesetzt ist → `elevated = true` an `GateCascadeEvaluator.evaluate()` uebergeben
- [ ] Die `GateCascadeEvaluator`-Klasse selbst aendert sich NICHT — sie benutzt bereits den `elevated`-Parameter; nur der Aufrufer wird erweitert
- [ ] Aufrufer-Klasse identifizieren (Glob nach `.evaluate(indicators, snapshot, ` suchen) und dort die ODER-Verknuepfung ergaenzen
- [ ] Unit-Test: `VPIN_ELEVATED`-Flag gesetzt, Regime-Confidence > 0.7 → `elevated = true`
- [ ] Unit-Test: kein `VPIN_ELEVATED`, Regime-Confidence > 0.7 → `elevated = false` (unveraendertes Verhalten)
- [ ] Integrationstest: Kompletter Pfad — Bar-Sequenz mit hohem einseitigem Volumen → VPIN_ELEVATED → GateCascadeEvaluator im elevated-Modus

### IndicatorSnapshotEntity-Erweiterung (odin-brain)

- [ ] `IndicatorSnapshotEntity` erhaelt neues Feld `private Double vpin` (nullable)
- [ ] `@Column(name = "vpin")` Annotation
- [ ] Konstruktor wird um `Double vpin`-Parameter erweitert
- [ ] Getter `getVwap()` und analogen `getVpin()` ergaenzen
- [ ] JavaDoc-Kommentar: "VPIN value [0,1] at this bar time; null when VPIN was disabled or during warmup"
- [ ] Alle Aufrufstellen des Konstruktors pruefen und `vpin`-Argument ergaenzen
- [ ] `toString()` wird um VPIN-Feld erweitert

### Flyway-Migration

- [ ] Neue Migrationsdatei: `V030__add_vpin_to_indicator_snapshot.sql`
- [ ] Inhalt: `ALTER TABLE odin.indicator_snapshot ADD COLUMN vpin DOUBLE PRECISION NULL;`
- [ ] Nullable, kein Default — bestehende Zeilen erhalten NULL (korrekt, da historische Snapshots keinen VPIN haben)
- [ ] Migration laeuft fehlerfrei durch (testen mit `mvn flyway:migrate` oder Backend-Start)

## Technische Details

### Geaenderte Klassen

| Klasse | Datei | Aenderung |
|--------|-------|-----------|
| `DataFlag` | `odin-api/.../model/DataFlag.java` | Neuer Wert `VPIN_ELEVATED` |
| `KpiEngine` | `odin-brain/.../kpi/KpiEngine.java` | VPIN-Flag-Setzung vor IndicatorResult-Konstruktion |
| `IndicatorSnapshotEntity` | `odin-brain/.../persistence/IndicatorSnapshotEntity.java` | Neues Feld `Double vpin`, Konstruktor, Getter |
| Aufrufer von `GateCascadeEvaluator.evaluate()` | Zu identifizieren (DecisionArbiter oder RulesEngine) | ODER-Verknuepfung fuer elevated-Flag |

### Neue Dateien

| Datei | Pfad | Beschreibung |
|-------|------|-------------|
| `V030__add_vpin_to_indicator_snapshot.sql` | `odin-persistence/src/main/resources/db/migration/` | Nullable VPIN-Spalte in indicator_snapshot |

### Flag-Setzung in KpiEngine.onSnapshot() (Schematisch)

```java
// compute vpin
double vpin = vpinCalculator != null ? vpinCalculator.onBar(latestBar) : Double.NaN;

// derive flags: start with snapshot flags, add VPIN_ELEVATED if warranted
Set<DataFlag> resultFlags = new HashSet<>(snapshot.flags());
if (!Double.isNaN(vpin) && vpin >= kpiProperties.vpin().elevatedThreshold()) {
    resultFlags.add(DataFlag.VPIN_ELEVATED);
}

IndicatorResult result = new IndicatorResult(
    ...,
    Set.copyOf(resultFlags),  // immutable copy
    null,
    vpin
);
```

### Aufrufer von GateCascadeEvaluator identifizieren

Per Grep nach `gateCascadeEvaluator.evaluate(` oder `.evaluate(indicators, snapshot` im odin-brain Modul suchen. Erwartet in `DecisionArbiter` oder einer Rules-Engine-Klasse. Dort die Logik:

```java
boolean elevated = (regimeConfidence >= 0.5 && regimeConfidence <= 0.7)
                 || indicators.flags().contains(DataFlag.VPIN_ELEVATED);
GateCascadeResult cascadeResult = gateCascadeEvaluator.evaluate(indicators, snapshot, elevated);
```

### Flyway-Migration V030

```sql
-- V030__add_vpin_to_indicator_snapshot.sql
ALTER TABLE odin.indicator_snapshot
    ADD COLUMN vpin DOUBLE PRECISION NULL;

COMMENT ON COLUMN odin.indicator_snapshot.vpin
    IS 'VPIN (Volume-Synchronized Probability of Informed Trading), range [0,1]; NULL when disabled or during warmup';
```

### IndicatorSnapshotEntity-Konstruktor nach Erweiterung (Auszug)

```java
public IndicatorSnapshotEntity(UUID runId, String instrumentId, Instant barTime,
                                Double ema9, Double ema21, ..., Double anchoredVwap,
                                Double vpin) {  // NEU
    // ...
    this.vpin = vpin;
}
```

## Konzept-Referenzen

- `docs/meta/userstories/ODIN-081_vpin/story.md` — Uebergeordnete Story; Gate-Integration-Konzept
- `docs/meta/userstories/ODIN-081b_vpin-calculator/story.md` — Abhaengige Story (IndicatorResult.vpin, VpinProperties)
- `docs/backend/architecture/06-rules-engine.md` — FSM-Zustaende, Decision-Loop, GateCascadeEvaluator-Aufruf
- `docs/backend/architecture/08-data-model.md` — indicator_snapshot-Tabelle, Zwei Schreibpfade, Flyway

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Abhaengigkeitsregeln; odin-persistence ist reine Infrastruktur (keine Domaenen-Logik)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `T:\codebase\its_odin\CLAUDE.md` — Datenbankzugriff: ausschliesslich `bash T:/codebase/its_odin/odin-db.sh "SQL"`, kein direkter psql

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api,odin-brain,odin-persistence`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (nicht anwendbar; Schwellenwerte kommen aus `VpinProperties`)
- [ ] Records fuer DTOs (keine neuen Records in dieser Story)
- [ ] ENUM statt String (`VPIN_ELEVATED` als Enum-Wert in `DataFlag`)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.kpi.vpin.*` (bereits in ODIN-081b eingerichtet)
- [ ] Port-Abstraktion: nicht anwendbar

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer VPIN-Flag-Setzungslogik: elevated, nicht elevated, NaN-Waechter
- [ ] Unit-Tests fuer elevated-Flag-Derivation beim GateCascadeEvaluator-Aufrufer: VPIN-Flag allein genuegt, ODER-Verknuepfung korrekt
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Bar-Sequenz mit einseitigem Volumen → VPIN_ELEVATED → `evaluate()` mit `elevated=true`
- [ ] Integrationstest: Geringer VPIN (balanced Volumen) → kein VPIN_ELEVATED → `evaluate()` mit `elevated=false`
- [ ] Testklassen-Namenskonvention: `VpinGateIntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank (falls zutreffend)
- [ ] Embedded-Postgres-Test (Zonky) fuer Flyway-Migration V030
- [ ] Test: `IndicatorSnapshotEntity` mit `vpin`-Wert speichern und laden → Wert korrekt persistiert
- [ ] Test: bestehende Zeilen ohne `vpin` bleiben NULL (Migration nicht destruktiv)
- [ ] Testklassen-Namenskonvention: `IndicatorSnapshotVpinIntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: Klassen, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn VPIN_ELEVATED und Regime-Confidence beide niedrig? Race Condition beim Flag-Setzen? Flyway-Kompatibilitaet mit Nullable?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review — Flag-Setzung korrekt (kein NaN-Flag), HashSet-Kopie korrekt, Flyway-Migration idempotent
- [ ] Dimension 2: Konzepttreue-Review — VPIN-Gate-Logik entspricht Konzept (Modifier, kein Hard-Gate); elevated-ODER-Logik korrekt
- [ ] Dimension 3: Praxis-Review — Was sind Konsequenzen wenn VPIN_ELEVATED sehr haeufig gesetzt wird? Wird Trading blockiert? Threshold-Kalibrierung sinnvoll?
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis wird waehrend der Arbeit live aktualisiert
- [ ] Pflichtabschnitte: Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### GateCascadeEvaluator selbst NICHT aendern

Der `GateCascadeEvaluator` ist sauber implementiert und nimmt bereits einen `elevated`-Boolean entgegen. Nur der AUFRUFER (vermutlich `DecisionArbiter`) muss erweitert werden. Das ist absichtlich — Separation of Concerns: Die Gate-Kaskade weiss nicht, WARUM elevated=true ist, nur DASS es so ist.

### Flags-Set aus Snapshot ist unveraenderlich

`MarketSnapshot.flags()` und `snapshot.flags()` liefern ein `Set.copyOf()`. Fuer die Flag-Erweiterung eine `new HashSet<>(snapshot.flags())` erstellen, `VPIN_ELEVATED` hinzufuegen, dann `Set.copyOf()` an `IndicatorResult` uebergeben.

### Flyway-Versionsnummer V030

Die letzte bekannte Migration ist V029 (`V029__allow_3m_bar_interval.sql`). Die naechste ist V030. Sicherstellen dass keine Luecken entstehen.

### Datenbankzugriff nur ueber das Shell-Skript

Fuer manuelle Tests der Migration gilt: `bash T:/codebase/its_odin/odin-db.sh "SELECT column_name FROM information_schema.columns WHERE table_name='indicator_snapshot'"`. Kein direkter psql-Aufruf.

### IndicatorSnapshotRepository-Aufrufe

Nach Konstruktor-Erweiterung der Entity alle Stellen suchen, wo `new IndicatorSnapshotEntity(...)` aufgerufen wird (vermutlich in einem `IndicatorSnapshotRepository` oder einem Service der KpiEngine-Ergebnisse persistiert). Diese Stellen muessen das `vpin`-Argument erhalten.
