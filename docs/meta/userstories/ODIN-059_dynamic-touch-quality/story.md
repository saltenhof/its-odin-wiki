# ODIN-059: Dynamic Touch Quality Scoring

**Modul:** odin-brain
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-058 (Spatial Touch Registry)

---

## Kontext

Im aktuellen System wird Touch-Quality zum Schreibzeitpunkt berechnet und im TouchEvent gespeichert. Geseedete Touches (aus Pivot-Daten) werden mit Zukunftsdaten bewertet (Lookahead), waehrend Live-Touches nur den aktuellen Bar sehen — das erzeugt eine systematische Quality-Asymmetrie. Geseedete Touches wirken "besser" als identische Live-Touches. Dieses Konzept eliminiert die Asymmetrie: TouchEvents speichern nur T=0-Fakten (was zum Zeitpunkt des Touch bekannt war), Quality wird bei jeder Abfrage dynamisch berechnet. Follow-Through (wie stark der Preis nach dem Touch reagierte) waechst natuerlich mit dem Alter des Touch — ein frischer Touch ist sofort actionable mit T=0-Metriken, ein aelterer Touch hat zusaetzlich Follow-Through-Informationen.

## Scope

**In Scope:**
- Neue Klasse `TouchQuality` in `de.its.odin.brain.sr` mit statischer Methode `computeDynamic()`
- T=0-Komponenten: Proximity (50%), Volume Signal (30%), Velocity Signal (20%)
- Follow-Through-Komponente mit Maturity-Blend (max 40% Gewicht bei HORIZON=3 Bars)
- Hilfsfunktionen: `computeProximity()`, `computeVolumeSignal()`, `computeVelocitySignal()`, `computeFollowThrough()`
- Integration in `SrLevelEngine` (bzw. `SupportLevelConfidence`): Quality pro Touch dynamisch berechnen statt aus gespeichertem Feld lesen
- Konfigurationsparameter in `SrProperties.TouchScoringProperties`
- Vereinfachung von `seedTouchesFromPivots()` — keine Quality-Berechnung mehr beim Seeding

**Out of Scope:**
- TouchRegistry-Architektur (ODIN-058 — bereits implementiert)
- Confidence-Rekalibrierung (ODIN-060 — Base+Boost, Tightness-Floor)
- Output Policy (ODIN-060)
- Aenderungen am TouchEvent-Record (bereits in ODIN-058 umgebaut)
- Aenderungen an der Touch-Detection-Logik (Proximity-Check unveraendert)

## Akzeptanzkriterien

### TouchQuality.computeDynamic()

- [ ] Statische Methode `computeDynamic(TouchEvent touch, List<Bar> oneMinBars, int currentBarIndex, double levelCenter, double atr1m)` → `double` in [0.0, 1.0]
- [ ] Berechnung von `age = currentBarIndex - touch.barIndex()`
- [ ] **T=0-Score** (immer verfuegbar):
  - Proximity: Gewicht 50% — `1.0 - clamp01(abs(touchPrice - levelCenter) / (2 * atr1m))`
  - Volume Signal: Gewicht 30% — `satExp(volumeRatio - 1.0, 2.0)` — saturierende Exponentialfunktion
  - Velocity Signal: Gewicht 20% — `clamp01(abs(priceVelocity) / (2 * atr1m))`
- [ ] **Follow-Through** (nur wenn `age >= 1`):
  - Berechnet aus Bars nach dem Touch bis `min(barIndex + FOLLOW_THROUGH_HORIZON, currentBarIndex)`
  - Misst maximale Entfernung des Preises vom Level nach dem Touch (Bounce-Staerke)
  - `clamp01(maxSeparation / (3 * atr1m))`
- [ ] **Maturity Blend:**
  - `maturityBlend = min(1.0, age / FOLLOW_THROUGH_HORIZON)`
  - `quality = (1.0 - maturityBlend * FOLLOW_THROUGH_MAX_WEIGHT) * t0Score + maturityBlend * FOLLOW_THROUGH_MAX_WEIGHT * followThrough`
  - Bei age=0 (Live-Touch): maturityBlend=0.0 → 100% T=0-Score → sofort actionable
  - Bei age >= HORIZON (3 Bars): maturityBlend=1.0 → 60% T=0 + 40% Follow-Through

### Asymmetrie-Elimination

- [ ] Live-Touch (age=0): Quality basiert zu 100% auf T=0-Metriken, kein Follow-Through, **kein Lag**
- [ ] Seeded Touch und Live Touch am selben barIndex, abgefragt zum selben currentBarIndex → **exakt die gleiche Quality**
- [ ] Touch mit age=30: Identische Quality wie Touch mit age=3 (HORIZON gecappt bei 3)
- [ ] Funktion ist deterministisch: gleiche Inputs → gleicher Output

### Hilfsfunktionen

- [ ] `computeProximity(double touchPrice, double levelCenter, double atr1m)` → double [0.0, 1.0]
  - Perfekter Touch am Center → 1.0
  - Weiter weg → linear sinkend
- [ ] `computeVolumeSignal(double volumeRatio)` → double [0.0, 1.0]
  - volumeRatio=1.0 (durchschnittlich) → Signal ≈ 0
  - Doppeltes Volumen → Signal ≈ 0.39
  - Hoher Spike → Signal → 1.0
- [ ] `computeVelocitySignal(double priceVelocity, double atr1m)` → double [0.0, 1.0]
  - Schnelle Annaeherung → hoeheres Signal
  - Normalisiert durch ATR
- [ ] `computeFollowThrough(List<Bar> afterBars, TouchEvent touch, double levelCenter, double atr1m)` → double [0.0, 1.0]
  - Maximale Entfernung des Close-Preises vom Level in den Folge-Bars
  - Normalisiert durch `3 * atr1m`
  - Leere afterBars → 0.0
- [ ] Mathematische Hilfsfunktionen `clamp01()` und `satExp()` — in existierender `SrMath` oder neuer Utility

### Integration in Engine / Confidence

- [ ] `SupportLevelConfidence` (oder `SrLevelEngine.scoreZones()`) berechnet Quality pro Touch dynamisch via `TouchQuality.computeDynamic()`
- [ ] `touchRegistry.countValidated()` nutzt dynamisch berechnete Quality statt gespeicherte Quality
  - Entweder: `countValidated()` erhaelt Callback/Funktionsreferenz fuer Quality-Berechnung
  - Oder: Abfrage erfolgt ueber `queryRange()` + externe Filterung in der Engine
  - Design-Entscheidung des Implementierers
- [ ] `seedTouchesFromPivots()` wird vereinfacht: nur rohes TouchEvent in Registry, **keine Quality-Berechnung** beim Seeding

### Konfiguration

- [ ] `SrProperties.TouchScoringProperties` als neues nested Record:
  - `int followThroughHorizonBars` (Default: 3)
  - `double followThroughMaxWeight` (Default: 0.40)
  - `double t0ProximityWeight` (Default: 0.50)
  - `double t0VolumeWeight` (Default: 0.30)
  - `double t0VelocityWeight` (Default: 0.20)
- [ ] Namespace: `odin.brain.sr.touch-scoring.*`
- [ ] Defaults in `application-brain.properties`
- [ ] Gewichte summieren sich zu 1.0 (Proximity + Volume + Velocity = 1.0)

### Konstanten

- [ ] `FOLLOW_THROUGH_HORIZON = 3` (Bars bis volle Reife — konfigurierbar)
- [ ] `FOLLOW_THROUGH_MAX_WEIGHT = 0.40` (Max 40% Follow-Through — konfigurierbar)
- [ ] `T0_PROXIMITY_WEIGHT = 0.50` (konfigurierbar)
- [ ] `T0_VOLUME_WEIGHT = 0.30` (konfigurierbar)
- [ ] `T0_VELOCITY_WEIGHT = 0.20` (konfigurierbar)
- [ ] Alle als `private static final` mit sprechenden Namen

## Technische Details

**Dateien (neu):**
- `odin-brain/src/main/java/de/its/odin/brain/sr/TouchQuality.java` — Statische Utility-Klasse mit `computeDynamic()` und Hilfsfunktionen

**Dateien (Umbau):**
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelEngine.java` — `seedTouchesFromPivots()` vereinfachen (keine Quality-Berechnung), `scoreZones()` mit dynamischer Quality
- `odin-brain/src/main/java/de/its/odin/brain/sr/SupportLevelConfidence.java` — behavioral.touchQAvg dynamisch berechnen
- `odin-brain/src/main/java/de/its/odin/brain/sr/TouchRegistry.java` — `countValidated()` muss mit dynamischer Quality arbeiten (ggf. Signaturanpassung)
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrProperties.java` — TouchScoringProperties hinzufuegen
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` — TouchScoring-Config einbinden

**Dateien (Properties):**
- `odin-brain/src/main/resources/application-brain.properties` — Defaults fuer touch-scoring

**Muster:**
- `TouchQuality` ist eine reine Utility-Klasse mit statischen Methoden — kein State, keine Instanzen
- Alle Berechnungen sind pure Functions (gleiche Inputs → gleicher Output)
- `satExp(x, k)` = `1.0 - exp(-k * max(0, x))` — saturierende Exponentialfunktion, Wertebereich [0, 1)

## Konzept-Referenzen

- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Vollstaendiges Konzept (Dynamic Scoring, T=0-Metriken, Follow-Through, Maturity-Blend)
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Abschnitt "Dynamic Quality Scoring: Berechnung zum Abfragezeitpunkt", vollstaendiger `computeDynamic()`-Pseudocode
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Abschnitt "T=0-Metriken im Detail", Proximity/Volume/Velocity-Formeln
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Abschnitt "Follow-Through (nur wenn verfuegbar)", `computeFollowThrough()`-Pseudocode
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Abschnitt "Wie die Asymmetrie verschwindet", Vergleichstabelle age vs. Quality-Zusammensetzung
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Abschnitt "Auswirkungen auf Seeding", vereinfachtes `seedTouchesFromPivots()`
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` — Abschnitt "Konfigurationsparameter (SrProperties)", TouchScoringProperties Record

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Namensregeln fuer `de.its.odin.brain.sr`
- `docs/backend/guardrails/development-process.md` — "Neue Geschaeftslogik → Unit-Test PFLICHT", "Risikoeinstufung HIGH fuer Trading-Logic"
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln (R1-R13), keine Magic Numbers, explizite Typen, Records fuer DTOs

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (alle Gewichte, Horizons, etc.)
- [ ] Records fuer DTOs (TouchScoringProperties als Record)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.sr.touch-scoring.*` fuer neue Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Test: `TouchQualityTest` — `computeDynamic()` mit Live-Touch (age=0): 100% T=0-Score
- [ ] Unit-Test: Touch mit age=1: T=0-Score + partielles Follow-Through
- [ ] Unit-Test: Touch mit age=3 (HORIZON): 60% T=0 + 40% Follow-Through
- [ ] Unit-Test: Touch mit age=30: identische Quality wie age=3 (HORIZON gecappt)
- [ ] Unit-Test: Seeded Touch und Live Touch am selben barIndex → identische Quality bei gleicher Abfrage-Zeit
- [ ] Unit-Test: `computeProximity()` — perfekter Touch (am Center) → 1.0
- [ ] Unit-Test: `computeProximity()` — Touch weit vom Center → nahe 0.0
- [ ] Unit-Test: `computeVolumeSignal()` — durchschnittliches Volumen (ratio=1.0) → ≈ 0
- [ ] Unit-Test: `computeVolumeSignal()` — doppeltes Volumen (ratio=2.0) → ≈ 0.39
- [ ] Unit-Test: `computeVolumeSignal()` — hoher Spike (ratio=5.0) → nahe 1.0
- [ ] Unit-Test: `computeVelocitySignal()` — schnelle Annaeherung → hohes Signal
- [ ] Unit-Test: `computeVelocitySignal()` — keine Bewegung (velocity=0) → 0.0
- [ ] Unit-Test: `computeFollowThrough()` — leere afterBars → 0.0
- [ ] Unit-Test: `computeFollowThrough()` — starke Separation → hoher Wert
- [ ] Unit-Test: `computeFollowThrough()` — Preis bleibt am Level kleben → niedriger Wert
- [ ] Unit-Test: Hoher Volume-Spike am Touch-Bar → hoehere T=0-Quality
- [ ] Unit-Test: Alle Ergebnisse in [0.0, 1.0] (keine Werte ausserhalb dank clamp01)
- [ ] Unit-Test: Registry-Eintraege sind immutable — kein Feld aendert sich nach Schreibvorgang
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendiger Scoring-Pfad: TouchRegistry → TouchQuality.computeDynamic() → SupportLevelConfidence → Zone-Confidence
- [ ] Integrationstest: Seeding + Live-Touch-Mix: geseedete und live erkannte Touches werden korrekt dynamisch bewertet
- [ ] Integrationstest: Quality wechselt ueber die Zeit: Touch bei barIndex=5, Abfrage bei barIndex=5 (age=0) vs. barIndex=8 (age=3) → Quality steigt durch Follow-Through
- [ ] Mindestens 1 Integrationstest der den Pfad TouchEvent-Erzeugung → Registry → Dynamic Quality → Confidence-Berechnung durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: TouchQuality-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei ATR=0? Was bei negativem barIndex? Numerische Stabilitaet von satExp bei extremen Werten? Verhalten wenn oneMinBars kuerzer als erwartet?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, ArrayIndexOutOfBounds bei subList(), Division-by-Zero bei ATR=0, numerische Praezision bei Gewichts-Summierung, Edge Cases bei leeren Bar-Listen"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche Gewichte (50/30/20), HORIZON=3, MAX_WEIGHT=0.40, Maturity-Blend-Formel, Follow-Through-Berechnung, satExp-Funktion, Proximity-Formel"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Quality-Stabilitaet bei unterschiedlichen ATR-Regimes, Verhalten bei Aktien mit niedrigem Volumen, Auswirkung auf bestehende Backtest-Ergebnisse"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Integration in countValidated(), satExp-Implementierung, Umgang mit fehlenden Bar-Daten)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Abhaengigkeit ODIN-058:** Diese Story setzt voraus, dass die TouchRegistry steht und TouchEvent bereits die neuen Felder hat (barIndex, touchPrice, priceVelocity, volumeRatio). Vor Beginn pruefen, dass ODIN-058 vollstaendig abgeschlossen ist.
- **satExp-Funktion:** `satExp(x, k) = 1.0 - exp(-k * max(0, x))`. Wertebereich [0, 1). Bei x=0 → 0, bei x→∞ → 1. Der Parameter k steuert die Steilheit. Fuer Volume Signal: k=2.0 (Default). Pruefen ob eine bestehende Math-Utility existiert oder ob `SrMath` erweitert werden muss.
- **clamp01:** `max(0.0, min(1.0, x))` — einfache Hilfsfunktion, moeglicherweise bereits in `SrMath` vorhanden.
- **Bar-Zugriff:** `computeFollowThrough()` braucht Zugriff auf Bars nach dem Touch-Bar. `oneMinBars.subList(touch.barIndex() + 1, lookEnd + 1)` — Vorsicht mit IndexOutOfBoundsException wenn oneMinBars kuerzer als erwartet ist. Defensive Programmierung mit `Math.min(lookEnd + 1, oneMinBars.size())`.
- **Integration in countValidated():** Die TouchRegistry.countValidated() aus ODIN-058 hat eine `minQuality`-Signatur. Mit dynamischer Quality muss die Berechnung ausgelagert werden — entweder die Registry bekommt einen `Function<TouchEvent, Double>`-Parameter, oder die Filterung passiert ausserhalb der Registry (z.B. in `SrLevelEngine`). Letzteres ist einfacher und sauberer.
- **Performance:** `computeDynamic()` wird pro Touch pro Abfrage aufgerufen. Bei typisch < 500 Touches pro Session und HORIZON=3 Bars ist das vernachlaessigbar. Kein Caching noetig.
- **Testdaten:** Fuer Unit-Tests eigene Bar-Listen konstruieren mit bekannten OHLCV-Werten. So lassen sich erwartete Quality-Werte exakt vorausberechnen.
- **Bestehende Quality-Berechnung:** Im aktuellen Code berechnet `SupportLevelConfidence` die behavioral-Komponente mit `touchQAvg` aus gespeicherten Quality-Werten. Diese Berechnung muss auf `TouchQuality.computeDynamic()` umgestellt werden. Den aktuellen Code in `SupportLevelConfidence.java` genau studieren.
