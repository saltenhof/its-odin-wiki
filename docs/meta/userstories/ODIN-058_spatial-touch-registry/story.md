# ODIN-058: Spatial Touch Registry + Simplified Reconciliation

**Modul:** odin-brain
**Phase:** 1
**Groesse:** XL
**Abhaengigkeiten:** Keine (Fundament fuer ODIN-059, ODIN-060, ODIN-061, ODIN-062)

---

## Kontext

Touches gehoeren aktuell zum Level (`SrLevel.touches`). Wenn ein Level durch Cluster-Center-Drift seine Position verschiebt oder geloescht wird, gehen alle zugehoerigen Live-Touches verloren — oder schlimmer: sie "wandern" mit dem Level an einen Preis, an dem sie nie stattfanden (Ghost Touches). Ein Level bei $41.95 behauptet 5 Touches zu haben, obwohl diese bei $41.80 stattfanden. Dieses Konzept fuehrt eine globale, raeumlich indizierte TouchRegistry ein. Touches gehoeren zum **Preis**, nicht zum Level. Levels fragen bei Bedarf ihre Preiszone ab. Reconciliation wird massiv vereinfacht: kein Touch-Transfer, keine Center-Glaettung, kein Split/Merge.

## Scope

**In Scope:**
- Neue Klasse `TouchRegistry` in `de.its.odin.brain.sr` mit $0.01-Bin-TreeMap
- Umbau von `TouchEvent` auf reines T=0-Record (Entfernung von `quality`, Hinzufuegen von `barIndex`, `priceVelocity`, `volumeRatio`)
- Entfernung von `touches`-Liste und `addTouch()` aus `SrLevel`
- Delegation von `SrLevel.validatedTouchCount()` an Registry
- Delegation von `SrZone.totalValidatedTouchCount()` an Registry (statt Aggregation ueber Level)
- Umbau von `detectTouches()` in `SrLevelEngine`: schreibt in TouchRegistry statt in Level
- Umbau von `seedTouchesFromPivots()` in `SrLevelEngine`: schreibt in TouchRegistry statt in Level
- Vereinfachte `reconcileSwingClusters()`: erweitertes Matching, harte Center-Updates, Grace Period, kein Split/Merge
- Neuer Pipeline-Schritt `enrichZonesFromRegistry()` zum Abfragen der touchCount-Werte pro Zone
- Konfigurationsparameter in `SrProperties.ReconciliationProperties`

**Out of Scope:**
- Dynamic Quality Scoring (ODIN-059 — TouchQuality.computeDynamic())
- Confidence-Rekalibrierung (ODIN-060 — Base+Boost, Tightness-Floor)
- Output Policy (ODIN-060 — minConf-Threshold, maxZones, Spatial Allocation)
- Band-API und Hybrid-Rendering (ODIN-062)
- Konsolidierungszonen-Erkennung (ODIN-061)
- Aenderungen am DBSCAN-Algorithmus (Clustering bleibt unveraendert)
- Aenderungen an der Touch-Detection-Logik (Proximity-Check bleibt, nur Schreibziel aendert sich)

## Akzeptanzkriterien

### TouchRegistry

- [ ] Neue Klasse `TouchRegistry` mit `NavigableMap<Integer, List<TouchEvent>>` ($0.01-Bins)
- [ ] Methode `record(TouchEvent touch)` — speichert Touch in zugehoerigem Preis-Bin
- [ ] Methode `queryRange(double lower, double upper)` — liefert alle Touches im Preisbereich via `TreeMap.subMap()` (inclusive bounds)
- [ ] Methode `countValidated(double lower, double upper, double minQuality)` — zaehlt Touches mit Quality >= Threshold im Bereich
- [ ] Private Hilfsmethode `priceToBin(double price)` — `Math.round(price / BIN_SIZE)` mit `BIN_SIZE = 0.01`
- [ ] Kein Ghost-Touch-Problem: Touch bei $41.80 → `queryRange(41.90, 42.00)` liefert ihn NICHT
- [ ] Touch bei $41.80 → `queryRange(41.75, 41.85)` liefert ihn
- [ ] Registry ist pro Pipeline-Instanz (Feld in `SrLevelEngine`)

### TouchEvent (Umbau)

- [ ] `TouchEvent` ist immutable Record mit **nur** T=0-Fakten:
  - `Instant timestamp` — Zeitpunkt des Touch
  - `int barIndex` — 1m-Bar-Index im Session-Kontext (**neu**)
  - `double touchPrice` — Preis am Touch-Punkt (ersetzt `price`)
  - `double barHigh`, `double barLow`, `double barClose` — OHLC des Touch-Bars
  - `long barVolume` — Volumen des Touch-Bars (ersetzt `volume`)
  - `double priceVelocity` — Preisaenderung in den letzten N Bars VOR dem Touch (**neu**)
  - `double volumeRatio` — Volumen des Touch-Bars / Durchschnittsvolumen (**neu**)
- [ ] Feld `quality` ist ENTFERNT (wird ab ODIN-059 dynamisch berechnet)
- [ ] Kein `isPending`-Flag, kein `effectiveLookaheadBars`

### SrLevel (Umbau)

- [ ] `touches`-Liste (Feld `List<TouchEvent> touches`) ENTFERNT
- [ ] Methode `addTouch(TouchEvent)` ENTFERNT
- [ ] Methode `validatedTouchCount()` delegiert an TouchRegistry: `registry.countValidated(center - band, center + band, VALIDATED_TOUCH_QUALITY_MIN)`
- [ ] Methode `averageTouchQuality()` delegiert an TouchRegistry (oder wird entfernt/verschoben — Design-Entscheidung des Implementierers)
- [ ] Methode `lastActivityTime()` — Anpassung an Registry-basierte Touch-Abfrage
- [ ] `SrLevel` erhaelt Referenz auf TouchRegistry (Konstruktor-Parameter oder Setter)
- [ ] `toString()` zeigt weiterhin Touch-Anzahl (ueber Registry-Query)

### SrZone (Umbau)

- [ ] `totalValidatedTouchCount()` aggregiert ueber Registry statt ueber `level.validatedTouchCount()`
- [ ] `averageTouchQuality()` aggregiert ueber Registry statt ueber `level.touches()`
- [ ] `touchPriceStdDev()` fragt Touch-Preise aus Registry statt aus Level-Touch-Listen
- [ ] SrZone erhaelt Referenz auf TouchRegistry

### SrLevelEngine (Umbau)

- [ ] Neues Feld `TouchRegistry touchRegistry` — wird pro Pipeline-Instanz erzeugt
- [ ] `detectTouches()` schreibt `touchRegistry.record(touch)` statt `level.addTouch(touch)`
- [ ] `seedTouchesFromPivots()` schreibt `touchRegistry.record(touch)` statt `newLevel.addTouch(touch)`
- [ ] `seedTouchesFromPivots()` erzeugt TouchEvent mit neuen Feldern (barIndex, priceVelocity, volumeRatio) — Quality wird **nicht** berechnet (wird in ODIN-059 dynamisch)
- [ ] Neuer Pipeline-Schritt `enrichZonesFromRegistry()` nach `fuseLevelsIntoZones()` — setzt touchCount pro Zone basierend auf Registry-Abfrage
- [ ] Pipeline-Flow-Reihenfolge:
  1. `updateOpeningRanges()` (unveraendert)
  2. `detectAndClusterSwingLevels()` (Reconciliation vereinfacht)
  3. `detectTouches()` (schreibt in Registry)
  4. `fuseLevelsIntoZones()`
  5. `enrichZonesFromRegistry()` (**neu**)
  6. `scoreZones()`
  7. `suppressOverdenseZones()`

### Reconciliation (Vereinfachung)

- [ ] **Erweitertes Matching:** `matchable ⟺ abs(newCenter - existingCenter) <= 2 * epsilon ODER interval-overlap > 0`
- [ ] **Harte Center-Updates:** `existing.updateCenter(newCluster.center())` — KEINE Glaettung (kein Alpha-Smoothing)
- [ ] **Grace Period:** Unmatched Level → `state = ORPHANED`, `orphanCycles++`
  - Bleibt in aktiver Level-Liste (Touch-Erkennung laeuft weiter)
  - Loeschung erst wenn: `orphanCycles >= maxOrphanCycles AND preisfern` (farGate: `abs(center - currentPrice) > farGateAtrMult * ATR`)
  - Bei Re-Match: `state = ACTIVE, orphanCycles = 0`
- [ ] **Split/Merge-Logik ENTFERNT** — zweiter Cluster wird neues Level, schwaecher werdendes Level trocknet ueber Grace Period aus
- [ ] **Keine Touch-Migration** — Touches bleiben in der Registry an ihrem Preis

### Konfiguration

- [ ] `SrProperties.ReconciliationProperties` als neues nested Record:
  - `double matchDistanceMultiplier` (Default: 2.0 — Vielfaches von epsilon)
  - `int maxOrphanCycles` (Default: 3)
  - `double farGateAtrMult` (Default: 3.0)
- [ ] Namespace: `odin.brain.sr.reconciliation.*`
- [ ] Defaults in `application-brain.properties`

### API-Verifikation

- [ ] `GET /api/v1/charts/IREN/sr-levels?tradingDate=2026-02-23` liefert touchCount-Werte die der Registry-Abfrage entsprechen
- [ ] touchCount-Werte koennen sich gegenueber vorher aendern (erwartetes Verhalten — keine Ghost Touches mehr)

## Technische Details

**Dateien (neu):**
- `odin-brain/src/main/java/de/its/odin/brain/sr/TouchRegistry.java`

**Dateien (Umbau):**
- `odin-brain/src/main/java/de/its/odin/brain/sr/TouchEvent.java` — Record-Felder aendern
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevel.java` — touches-Liste entfernen, Registry-Delegation
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrZone.java` — Registry-Delegation
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelEngine.java` — TouchRegistry-Feld, detectTouches/seedTouches/reconciliation Umbau, enrichZonesFromRegistry() neu
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrProperties.java` — ReconciliationProperties hinzufuegen
- `odin-brain/src/main/java/de/its/odin/brain/sr/SupportLevelConfidence.java` — behavioral.touchCount aus Registry
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` — Reconciliation-Config einbinden

**Dateien (Test-Anpassungen):**
- Alle existierenden Tests die TouchEvent, SrLevel.addTouch(), SrLevel.touches() oder SrZone.totalValidatedTouchCount() verwenden muessen angepasst werden

**Muster:**
- `TouchRegistry` ist eine einfache POJO-Klasse (kein Spring Bean) — wird per Pipeline-Instanz erzeugt
- `SrLevel` und `SrZone` erhalten entweder eine Registry-Referenz oder die relevanten Methoden werden zu Engine-Level verschoben (Design-Entscheidung des Implementierers)
- `BIN_SIZE = 0.01` ist als `private static final` Konstante definiert (US-Equity Tick Size)

**Uebergangsphase Quality:**
Da ODIN-059 (Dynamic Quality Scoring) noch nicht existiert, muss in dieser Story eine **Uebergangsloesung** fuer `countValidated()` und Quality-basierte Abfragen getroffen werden. Optionen:
1. `countValidated()` zaehlt vorerst alle Touches (minQuality = 0.0), Quality-Filterung kommt mit ODIN-059
2. Oder eine einfache statische Quality-Berechnung basierend auf T=0-Metriken (Proximity + Volume) als Platzhalter

Design-Entscheidung liegt beim Implementierer — Hauptsache die Registry-Architektur steht.

## Konzept-Referenzen

- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Vollstaendiges Konzept (Spatial Touch Registry, Reconciliation-Vereinfachung, Ghost-Touch-Elimination)
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Abschnitt "Architektur: TouchRegistry (neu)", Code-Beispiel fuer Registry-Klasse
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Abschnitt "Was bleibt von reconcileSwingClusters()", Punkte 1-4
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Abschnitt "Auswirkungen auf andere Komponenten", Tabelle
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Abschnitt "Implementierung im Pipeline-Flow", Reihenfolge
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Abschnitt "Konfigurationsparameter", ReconciliationProperties
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Abschnitt "TouchEvent: Immutable, nur T=0-Fakten" (neue Record-Felder: barIndex, priceVelocity, volumeRatio; entfernt: quality, isPending)

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Namensregeln fuer `de.its.odin.brain.sr`
- `docs/backend/guardrails/development-process.md` — "Neue Geschaeftslogik → Unit-Test PFLICHT", "Risikoeinstufung HIGH fuer Trading-Logic"
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln (R1-R13), Port-Abstraktion, Records fuer DTOs, keine Magic Numbers

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (BIN_SIZE, VALIDATED_TOUCH_QUALITY_MIN, etc.)
- [ ] Records fuer DTOs (TouchEvent bleibt Record, ReconciliationProperties als Record)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.sr.reconciliation.*` fuer neue Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Test: `TouchRegistryTest` — `record()` + `queryRange()` mit verschiedenen Preisbereichen
- [ ] Unit-Test: Touch bei $41.80 → Query $41.75–$41.85 → enthaelt den Touch
- [ ] Unit-Test: Touch bei $41.80 → Query $41.90–$42.00 → enthaelt den Touch NICHT (kein Ghost Touch)
- [ ] Unit-Test: `countValidated()` zaehlt korrekt nach Quality-Threshold
- [ ] Unit-Test: Leere Registry → `queryRange()` liefert leere Liste
- [ ] Unit-Test: Mehrere Touches im gleichen Bin → alle zurueckgegeben
- [ ] Unit-Test: SrLevel ohne eigene Touch-Liste — delegiert korrekt an Registry
- [ ] Unit-Test: SrZone.totalValidatedTouchCount() aggregiert ueber Registry
- [ ] Unit-Test: Reconciliation — erweitertes Matching (2*epsilon Distanz)
- [ ] Unit-Test: Reconciliation — erweitertes Matching (Intervall-Overlap)
- [ ] Unit-Test: Reconciliation — harte Center-Updates (kein Smoothing)
- [ ] Unit-Test: Grace Period — Orphaned Level bleibt aktiv, Touch-Erkennung laeuft
- [ ] Unit-Test: Grace Period — Orphaned Level nach 3 Zyklen + preisfern → geloescht
- [ ] Unit-Test: Grace Period — Re-Match setzt orphanCycles auf 0
- [ ] Unit-Test: Seeded Touch + Live Touch am selben Preis → beide in Registry, kein Konflikt
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer KPI-Snapshot-Daten (keine Spring-Kontexte noetig)

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendiger Pipeline-Durchlauf mit `SrLevelEngine` und realer `TouchRegistry`
- [ ] Integrationstest: Level driftet $41.80 → $41.95 → touchCount aendert sich (Registry-basiert, kein Transfer)
- [ ] Integrationstest: Level driftet $41.95 → $41.80 zurueck → sieht die Original-Touches wieder
- [ ] Integrationstest: Zwei Level mit ueberlappenden Bands → teilen sich Touches korrekt
- [ ] Integrationstest: Registry nach voller Session-Simulation: alle Touches vorhanden, keine Duplikate
- [ ] Mindestens 1 Integrationstest der den Pfad Snapshot → DBSCAN → Reconciliation → TouchDetection → Registry → Zone-Enrichment durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: TouchRegistry-Code, SrLevel-Umbau, Reconciliation-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Bin-Grenzwerte ($41.005 — in welchen Bin?), leere Bins, TreeMap-Verhalten bei extremen Preisen, Floating-Point-Probleme bei Bin-Berechnung, Orphaned Level mit Touches in naher Zone
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, TreeMap-Boundary-Probleme, Floating-Point-Praezision bei Bin-Berechnung, Performance bei grossen Registries, korrekte SubMap-Boundaries"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `specs/sr-engine-v2/04-concept-reconciliation-drift.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche Registry-Architektur, Reconciliation-Vereinfachung, Grace-Period-Semantik, Entfernung von Split/Merge, Bin-Groesse, Matching-Kriterien"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Registry-Memory-Wachstum ueber den Tag, Verhalten bei Aktien mit Preisen > $500 (Bin-Anzahl), Thread-Safety bei zukuenftiger Parallelisierung"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. wie SrLevel Registry-Referenz erhaelt, Uebergangsloesung fuer Quality, Umgang mit SrZone-Delegation)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **XL-Umfang:** Diese Story ist bewusst XL weil sie das fundamentale Datenmodell umstellt. SrLevel, SrZone, SrLevelEngine, TouchEvent und SupportLevelConfidence sind alle betroffen. Gruendliches Verstaendnis der Bestandsarchitektur ist Voraussetzung.
- **Bestehende Tests:** Es gibt existierende Tests fuer SrLevel, SrZone, SrLevelEngine, SupportLevelConfidence etc. Diese werden durch den TouchEvent-Umbau und die Entfernung von `addTouch()` brechen. Alle betroffenen Tests muessen angepasst werden — das ist erwarteter Aufwand.
- **TouchEvent-Feldnamen:** Das Feld `price` wird zu `touchPrice` umbenannt (Konsistenz mit Konzept). Das Feld `volume` wird zu `barVolume` (Klarheit). Alle Verwendungsstellen anpassen.
- **Quality-Uebergang:** Da `quality` aus TouchEvent entfernt wird aber ODIN-059 noch nicht existiert, braucht `countValidated()` in der TouchRegistry eine Zwischenloesung. Empfehlung: vorerst alle Touches zaehlen (minQuality-Parameter beibehalten fuer spaetere Integration, aber mit 0.0 aufrufen). Alternativ eine simple statische T=0-Quality als Platzhalter. Dokumentiere die Entscheidung im protocol.md.
- **Reconciliation-Vereinfachung:** Die aktuelle `reconcileSwingClusters()` hat moeglicherweise bereits Matching-Logik. Vergleiche den bestehenden Code sorgfaeltig mit dem Konzept und entscheide was bleibt, was vereinfacht und was entfernt wird.
- **Orphan-State:** SrLevel braucht moeglicherweise ein neues Feld `orphanCycles` und einen State (`ACTIVE`/`ORPHANED`). Das existiert aktuell nicht — muss hinzugefuegt werden.
- **priceVelocity/volumeRatio-Berechnung:** Fuer `detectTouches()` muessen diese T=0-Metriken berechnet werden. Velocity = Preisaenderung ueber die letzten N Bars (z.B. 3-5 Bars), VolumeRatio = Bar-Volume / gleitender Durchschnitt. Die genaue Berechnungslogik steht in `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md`, Abschnitt "T=0-Metriken im Detail".
- **Pipeline-Reihenfolge:** Die Einfuegung von `enrichZonesFromRegistry()` nach `fuseLevelsIntoZones()` und vor `scoreZones()` ist wichtig — Scoring braucht die Registry-basierten touchCounts.
- **Bin-Groesse $0.01:** Entspricht der Tick-Size fuer US-Equities. Bei Instrumenten mit anderer Tick-Size koennte dies spaeter konfigurierbar werden (Phase 2). Vorerst fest.
