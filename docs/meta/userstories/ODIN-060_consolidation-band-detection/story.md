# ODIN-060: Consolidation Band Detection

**Modul:** odin-brain
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-058 (Spatial Touch Registry)
**Konzept-Referenz:** `specs/sr-engine-v2/02-concept-overdense-consolidation.md`

---

## Kontext

Die aktuelle S/R-Engine erzeugt in overdensen Preiszonen mehrere eng beieinanderliegende Cluster-Linien (z.B. 4 Linien in $0.37 Spanne bei IREN 23.02). NMS kann diese nicht sauber bereinigen, weil die Abstaende knapp ueber dem NMS-Radius liegen. Die Konsequenz: Downstream-Engines (LLM, Rules) sehen 4 separate Support/Resistance-Level statt einer kohaerenten Konsolidierungszone. Das verfaelscht Signale — ein Trader wuerde an der Lower-Line kaufen und an der Upper-Line shorten, obwohl der gesamte Bereich "Chop" ist.

Diese Story implementiert die Dichte-basierte Erkennung von Konsolidierungszonen ueber die Spatial Touch Registry (ODIN-058). Statt Cluster-Linien nachtraeglich zu gruppieren, wird die Touch-Dichte direkt gescannt. Das Ergebnis ist ein **ConsolidationBand** — ein Band-Objekt, das die gesamte Chop-Zone als eine Einheit repraesentiert.

## Scope

**In Scope:**
- `ConsolidationBand` Record in `de.its.odin.brain.sr`
- Dichte-basierter Sliding-Window-Scan auf der TouchRegistry
- POC-Berechnung (touch-quality-gewichteter Schwerpunkt)
- Ersetzungslogik: Cluster-Linien innerhalb eines Bands werden entfernt, Band-Objekt wird eingefuegt
- Monotones Band-Wachstum (neue Touches erweitern, schrumpfen nie)
- Statische Level (VWAP, OR) innerhalb des Bands bleiben separat erhalten
- `ConsolidationProperties` in `SrProperties` als `@ConfigurationProperties`-Record
- Pipeline-Integration: `detectConsolidationBands()` + `replaceClusterZonesWithBands()` nach NMS, vor Output Policy

**Out of Scope:**
- Frontend-Visualisierung der Bands (das ist ODIN-063)
- Aenderungen an der TouchRegistry selbst (ODIN-058)
- Aenderungen an NMS (suppressOverdenseZones) — NMS bleibt unveraendert
- Aenderungen an der Output Policy (Konzept 5 / ODIN-062)
- Downstream-Aenderungen in LLM-Analyst oder Rules Engine
- Breakout-Detection auf Bands

## Akzeptanzkriterien

### ConsolidationBand Record

- [ ] Record `ConsolidationBand` mit den Feldern: `bottom` (double), `top` (double), `poc` (double), `totalTouches` (int), `avgTouchQuality` (double), `firstTouch` (Instant), `lastTouch` (Instant)
- [ ] Record ist immutable (standard Record-Semantik)
- [ ] `spread()` Convenience-Methode: `top - bottom`

### Dichte-Scan-Algorithmus

- [ ] Sliding-Window-Scan ueber den relevanten Preisbereich (±`scanRangeAtrMult` × ATR um aktuellen Preis)
- [ ] Scan-Fensterbreite: `min(consolAtrMult × atr5m, consolPctCap × currentPrice)` — Default: 0.75 × ATR, Cap 1.5%
- [ ] Step-Size: 25% der Fensterbreite (75% Overlap zwischen aufeinanderfolgenden Fenstern)
- [ ] Dichte-Schwelle: `minTouchesPerWindow` (Default: 5) — nur validierte Touches (quality >= 0.50) zaehlen
- [ ] Zusammenhaengende Fenster mit Dichte >= Schwelle werden zu einer DenseRegion verbunden
- [ ] minSpread-Filter: DenseRegion wird nur zum Band wenn `spread >= minSpreadAtrMult × atr5m` (Default: 0.3 × ATR)

### POC-Berechnung

- [ ] POC = touch-quality-gewichteter Schwerpunkt: `sum(touchPrice × quality) / sum(quality)`
- [ ] Fallback wenn keine Touches vorhanden: geometrische Mitte `(bottom + top) / 2`
- [ ] POC liegt immer zwischen bottom und top (clamped)

### Band ersetzt Cluster-Linien

- [ ] Alle Cluster-Zonen (type=SWING_CLUSTER) deren Center innerhalb des Bands liegt werden entfernt
- [ ] Das Band wird als einzelne Zone exportiert mit: `center=POC`, `band=(top-bottom)/2`, `totalTouches=kumuliert`, `sources=CONSOLIDATION`
- [ ] Statische Level (VWAP, OR) deren Center innerhalb des Bands liegt bleiben als separate Zonen erhalten
- [ ] Wenn ein Band 0 Cluster-Linien ersetzt (nur statische Level im Band), wird kein Band erzeugt

### Monotones Band-Wachstum

- [ ] Ein bestehendes Band darf nur wachsen: `newBottom <= existingBottom`, `newTop >= existingTop`
- [ ] Wenn der Scan ein kleineres Band liefert als das bestehende, wird das bestehende beibehalten
- [ ] POC und totalTouches werden trotzdem aktualisiert (nur die Grenzen sind monoton)

### Konfiguration

- [ ] `ConsolidationProperties` als verschachtelter Record in `SrProperties`
- [ ] Namespace: `odin.brain.sr.consolidation.*`
- [ ] Parameter mit Defaults:
  - `consolAtrMult`: 0.75
  - `consolPctCap`: 0.015 (1.5%)
  - `minTouchesPerWindow`: 5
  - `minSpreadAtrMult`: 0.3
  - `scanRangeAtrMult`: 3.0
- [ ] Alle Parameter in `application.properties` mit Defaults

### Pipeline-Integration

- [ ] `detectConsolidationBands()` wird aufgerufen NACH `suppressOverdenseZones()` (NMS)
- [ ] `replaceClusterZonesWithBands()` wird aufgerufen direkt danach
- [ ] Beide Schritte VOR `applyOutputPolicy()`
- [ ] Pipeline-Reihenfolge:
  ```
  7. suppressOverdenseZones()        (NMS — unveraendert)
  8. detectConsolidationBands()      (NEU)
  9. replaceClusterZonesWithBands()  (NEU)
  10. applyOutputPolicy()            (unveraendert)
  ```

### API-Verifikation

- [ ] Die API-Response (SrSnapshot-DTO) enthaelt Konsolidierungszonen mit `band`-Werten > 0 und `sources=CONSOLIDATION`
- [ ] Backtest mit IREN 23.02 liefert: 1 ConsolidationBand statt 4 Cluster-Linien im Bereich $41.65–$42.10
- [ ] POC des Bands liegt nahe $42.04 (touch-gewichteter Schwerpunkt, nicht geometrische Mitte)
- [ ] Gesamte totalTouches des Bands ≈ 60

### Erwartetes Ergebnis fuer IREN 23.02

**Vorher (4 Cluster-Linien):**
```
41.703  touchCount=5   confidence=0.309  CLUSTER
41.815  touchCount=3   confidence=0.369  CLUSTER
41.929  touchCount=11  confidence=0.320  CLUSTER
42.073  touchCount=41  confidence=0.322  CLUSTER
```

**Nachher (1 ConsolidationBand):**
```
Band: bottom~41.65, top~42.10, poc~42.04
      totalTouches~60, type=CONSOLIDATION
```

4 Linien werden zu 1 Band-Objekt komprimiert. Klar kommuniziert: "Chop-Zone, handle den Breakout."

## Technische Details

**Neue Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/sr/ConsolidationBand.java` — Record
- `odin-brain/src/main/java/de/its/odin/brain/sr/ConsolidationBandDetector.java` — Dichte-Scan + POC-Berechnung + Band-Ersetzung
- `odin-brain/src/main/java/de/its/odin/brain/sr/DenseRegion.java` — internes Hilfs-Record fuer den Scan (bottom, top, touchCounts)

**Geaenderte Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelEngine.java` — Pipeline-Integration (Aufruf der neuen Schritte 8 + 9)
- `odin-brain/src/main/java/de/its/odin/brain/config/SrProperties.java` — `ConsolidationProperties` als verschachtelter Record
- `odin-brain/src/main/resources/application.properties` — Defaults fuer `odin.brain.sr.consolidation.*`

**Patterns:**
- `ConsolidationBandDetector` ist ein reines POJO (kein Spring Bean), instanziiert pro Pipeline durch PipelineFactory
- Nutzt `TouchRegistry` (aus ODIN-058) fuer den Dichte-Scan via `countValidated()` und `queryRange()`
- `SrLevelEngine` haelt eine Referenz auf `ConsolidationBandDetector` und ruft die beiden neuen Schritte in der Pipeline-Methode auf

**Abhaengigkeiten:**
- `TouchRegistry` (ODIN-058) — fuer `countValidated(lower, upper, minQuality)` und `queryRange(lower, upper)`
- `SrProperties.ConsolidationProperties` — fuer Konfigurationsparameter
- Bestehende Zone-Typen und SrSnapshot-Struktur

## Konzept-Referenzen

- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` — **gesamtes Dokument**: Problemstellung, Gemini-Kritik, Loesungsansatz, Architektur, Algorithmus, Pipeline-Flow, Konfiguration, Tests
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` — Abschnitt "Architektur: ConsolidationBand (neuer Zone-Typ)": Record-Definition
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` — Abschnitt "Erkennung: Touch-Dichte-Scan auf der Registry": Algorithmus mit Pseudocode
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` — Abschnitt "POC-Berechnung (Point of Control)": Gewichtete Schwerpunkt-Formel
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` — Abschnitt "Integration: Bands ersetzen Cluster-Linien innerhalb der Zone": Ersetzungslogik
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` — Abschnitt "Konfigurationsparameter (SrProperties)": Defaults und Namespace
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Abschnitt "Architektur: TouchRegistry (neu)": TouchRegistry-API (`countValidated`, `queryRange`)
- `specs/sr-engine-v2/05-concept-output-quality.md` — Abschnitt "Pipeline-Integration": Einordnung von Konsolidierung VOR Output Policy

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen: neue Klassen in `de.its.odin.brain.sr`
- `docs/backend/guardrails/development-process.md` — Risikoeinstufung HIGH fuer Trading-Logic, ChatGPT-Sparring Pflicht, Gemini 3-Dimensionen-Review Pflicht
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln (R1-R13): kein `var`, keine Magic Numbers, Records fuer DTOs, JavaDoc, Port-Abstraktion
- `T:\codebase\its_odin\CLAUDE.md` — Konfiguration (CSpec): `@ConfigurationProperties` als Record, Namespace `odin.brain.sr.consolidation.*`

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs (`ConsolidationBand`, `DenseRegion`)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.sr.consolidation.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: 4 Cluster in $0.37 Spanne mit 60 Touches --> ConsolidationBand erkannt
- [ ] Unit-Test: 2 Level mit $2 Abstand --> kein Band (Dichte-Luecke dazwischen)
- [ ] Unit-Test: Band-POC liegt beim touch-gewichteten Schwerpunkt (nicht geometrische Mitte)
- [ ] Unit-Test: VWAP innerhalb Band --> bleibt als separate Zone erhalten
- [ ] Unit-Test: Band waechst monoton -- neuer Touch am Rand erweitert, schrumpft nie
- [ ] Unit-Test: Fruehe Session (< 5 Touches im Scan-Fenster) --> kein Band, normale Cluster-Ausgabe
- [ ] Unit-Test: Band-Stabilitaet -- 10 aufeinanderfolgende Snapshots --> POC bewegt sich um < 0.02
- [ ] Unit-Test: Band-Zone hat type=CONSOLIDATION, enthaelt bottom/top/poc
- [ ] Unit-Test: minSpread-Filter -- DenseRegion mit Spread < 0.3 ATR wird nicht zum Band
- [ ] Unit-Test: Scan-Bereich ist begrenzt auf +-scanRangeAtrMult * ATR
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer TouchRegistry

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendige Pipeline mit realer TouchRegistry, realem ConsolidationBandDetector und realem SrLevelEngine -- End-to-End vom Snapshot bis zur finalen Zone-Ausgabe
- [ ] Integrationstest: IREN-23.02-aehnliches Setup (4 Cluster, 60 Touches) --> verifikation dass genau 1 Band entsteht
- [ ] Integrationstest: Mixed Setup (Cluster + VWAP im Band-Bereich) --> VWAP ueberlebt, Cluster werden ersetzt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: ConsolidationBandDetector-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei ATR nahe 0? Ueberlappende Bands? Band das den gesamten Preisbereich umfasst? Leere TouchRegistry?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, Off-by-One bei Sliding-Window-Grenzen, Performance bei grossen Touch-Registries, Floating-Point-Probleme bei Vergleichen"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `specs/sr-engine-v2/02-concept-overdense-consolidation.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche: ConsolidationBand-Felder, Scan-Parameter und Defaults, POC-Berechnung, Ersetzungslogik (Cluster raus, statische Level bleiben), monotones Wachstum, Pipeline-Reihenfolge"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Was passiert bei Instrumenten mit sehr geringer Liquiditaet (wenige Touches)? Was bei extrem volatilen Tagen mit sehr breiter ATR? Mehrere ueberlappende Bands?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. DenseRegion als internes Hilfsformat, Band-Identitaet ueber Sessions, Clamping des POC)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **ODIN-058 muss fertig sein.** Diese Story setzt die `TouchRegistry` mit `countValidated()` und `queryRange()` voraus. Ohne funktionsfaehige Registry ist kein Dichte-Scan moeglich.
- **Floating-Point-Sorgfalt:** Der Sliding-Window-Scan arbeitet mit `double`-Arithmetik. Step-Size und Fenster-Grenzen muessen sauber berechnet werden — Off-by-One oder Rundungsfehler koennen dazu fuehren, dass Touches an Fenster-Grenzen doppelt oder gar nicht gezaehlt werden.
- **Monotones Wachstum:** Das Band darf nur wachsen. Das bedeutet, dass ein vorheriges Band ueber Snapshots hinweg gespeichert und mit dem neuen Scan-Ergebnis verglichen werden muss. Die Band-Identitaet muss ueber die Preis-Position bestimmt werden (nicht ueber eine ID).
- **DenseRegion ist kein API-Typ.** Es ist ein internes Hilfsformat fuer den Scan-Algorithmus. Es wird nach dem Scan zu `ConsolidationBand` konvertiert und verschwindet dann.
- **VWAP bleibt:** Wenn ein VWAP-Level mitten im Band liegt, wird es NICHT vom Band absorbiert. Es bleibt als separate Zone mit eigenem Typ erhalten. Das ist eine explizite Design-Entscheidung aus dem Konzept.
- **Performance:** Der Scan-Bereich ist auf +-3 ATR begrenzt. Bei ATR = $0.25 sind das $1.50 in jede Richtung. Mit einer Step-Size von $0.047 (0.25 × 0.75 / 4) ergibt das ca. 64 Scan-Schritte pro Snapshot — unkritisch.
- **Kein Breakout-Detection:** Das Band sagt nur "hier ist Chop". Ob der Preis das Band nach oben oder unten durchbricht, ist Aufgabe der Rules Engine — nicht Teil dieser Story.
