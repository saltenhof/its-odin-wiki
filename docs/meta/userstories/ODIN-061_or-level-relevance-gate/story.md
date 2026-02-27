# ODIN-061: OR-Level Relevance Gate

**Modul:** odin-brain
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-058 (Spatial Touch Registry)
**Konzept-Referenz:** `specs/sr-engine-v2/01-concept-or-level-flooding.md`

---

## Kontext

Bei IREN am 23.02.2026 belegen 5 von 12 Ausgabe-Zonen Opening-Range-Level ($38.96–$40.47). Drei davon haben 0 Touches und liegen 6–8% unter der aktuellen Handelszone um $42.00. Diese Level sind irrelevant — sie verschwenden Slots, die fuer Cluster-Level mit echten Touches gebraucht werden. Gleichzeitig koennen OR-Level spaeter relevant werden, wenn der Preis zu ihnen zurueckkehrt. Ein statisches Entfernen waere also falsch.

Diese Story implementiert ein dynamisches Relevanz-Gate fuer OR-Level mit drei Status (PROTECTED, ACTIVE, PARKED). Die Statusbestimmung erfolgt dynamisch ueber die TouchRegistry und Preis-Distanz mit Hysterese-Schwellen, um Flickering zu vermeiden. PARKED-Level werden komplett aus der Pipeline entfernt. Kein Sonder-Quota, kein keepAlways — OR-Level konkurrieren gleichberechtigt mit Cluster-Leveln ueber die globale Output Policy (Konzept 5).

## Scope

**In Scope:**
- `OrRelevanceStatus` Enum (PROTECTED, ACTIVE, PARKED)
- Dynamische Statusbestimmung pro OR-Level (TouchRegistry-Abfrage + Preis-Distanz mit Hysterese)
- Hysterese-Gate mit asymmetrischen Schwellen (Flickering-Schutz)
- MID-Level: konditionale Generierung gegen Daily-ATR-Estimate
- PARKED-Level komplett aus Pipeline entfernen (vor Scoring)
- `OrRelevanceProperties` in `SrProperties` als `@ConfigurationProperties`-Record
- Pipeline-Integration: `computeOrRelevanceStatus()` nach Level-Erzeugung, vor Zone-Fusion

**Out of Scope:**
- Aenderungen an der TouchRegistry selbst (ODIN-058)
- Aenderungen an NMS — OR-Level haben KEINEN keepAlways-Schutz
- Aenderungen an der Output Policy (Konzept 5 / ODIN-062)
- Sonder-Quota fuer OR-Level (explizit verworfen im Konzept)
- Frontend-Visualisierung (visueller Effekt entsteht automatisch durch weniger Zonen)

## Akzeptanzkriterien

### OrRelevanceStatus Enum

- [ ] Enum `OrRelevanceStatus` mit drei Werten: `PROTECTED`, `ACTIVE`, `PARKED`
- [ ] Definiert in `de.its.odin.brain.sr`

### Dynamische Statusbestimmung

- [ ] Status wird pro OR-Level bei jedem Snapshot dynamisch berechnet — NICHT als statisches Flag gesetzt
- [ ] PROTECTED: `registry.countValidated(lower, upper, minQuality) > 0` — Level hat mindestens einen validierten Touch
- [ ] ACTIVE: 0 validierte Touches UND Preis-Distanz <= Hysterese-Schwelle (preisnah)
- [ ] PARKED: 0 validierte Touches UND Preis-Distanz > Hysterese-Schwelle (preisfern)
- [ ] Bei Uebergang zu PROTECTED (Touch kommt hinzu): Status aendert sich sofort, keine Verzoegerung

### Hysterese-Gate (Flickering-Schutz)

- [ ] Asymmetrische Schwellen:
  - PARKED --> ACTIVE (Aktivierung): `activateThreshold = min(activateAtrMult × atr5m, activatePctCap × currentPrice)` — Default: 4.0 ATR, Cap 3%
  - ACTIVE --> PARKED (Parken): `parkThreshold = min(parkAtrMult × atr5m, parkPctCap × currentPrice)` — Default: 5.0 ATR, Cap 4%
- [ ] Dead Zone zwischen activateThreshold und parkThreshold: kein Zustandswechsel, egal in welche Richtung
- [ ] Vorheriger Status muss pro OR-Level gespeichert werden (fuer korrekte Hysterese-Entscheidung)
- [ ] Initialer Status (erster Snapshot): ACTIVE wenn innerhalb activateThreshold, sonst PARKED

### MID-Level: Konditionale Generierung

- [ ] OR_MID wird nur erzeugt wenn: `OR_range <= midRangeMaxDailyAtrFraction × dailyAtrEstimate` — Default: 0.30
- [ ] Daily-ATR-Schaetzung (Proxy): `max(atr5m × sqrt(78), sessionHighSoFar - sessionLowSoFar)`
- [ ] Die 78 ist die Anzahl der 5-Minuten-Bars in einer RTH-Session (6.5h)
- [ ] Wenn OR-Range zu breit relativ zum Tagesverhalten: kein MID-Level, nur HIGH und LOW
- [ ] Breite OR mit Beispiel: Daily ATR $1.50, midRangeMaxDailyAtrFraction=0.30 --> MID nur bei OR_range <= $0.45

### Kein Sonder-Quota, kein keepAlways

- [ ] PROTECTED und ACTIVE OR-Level gehen als normale Kandidaten durch NMS und Scoring
- [ ] Kein `keepAlways` Flag fuer OR-Level — NMS merged normal (OR + VWAP auf gleichem Preis --> NMS darf mergen)
- [ ] Kein `unconfirmedOrMax` Parameter — die Output Policy (Konzept 5) reguliert natuerlich ueber Confidence
- [ ] ACTIVE OR-Level (0 Touches) bekommen nur Base Confidence (niedrig) und werden natuerlich runtergerankt

### PARKED-Level-Entfernung

- [ ] PARKED OR-Level werden KOMPLETT aus der Pipeline entfernt (nicht nur ausgeblendet)
- [ ] Entfernung findet VOR `fuseLevelsIntoZones()` statt — PARKED Level erscheinen nie als Zone
- [ ] Wenn der Preis spaeter zu einem geparkten Level zurueckkehrt: PARKED --> ACTIVE Transition ueber Hysterese

### Konfiguration

- [ ] `OrRelevanceProperties` als verschachtelter Record in `SrProperties`
- [ ] Namespace: `odin.brain.sr.or-relevance.*`
- [ ] Parameter mit Defaults:
  - `activateAtrMult`: 4.0
  - `activatePctCap`: 0.03 (3%)
  - `parkAtrMult`: 5.0
  - `parkPctCap`: 0.04 (4%)
  - `midRangeMaxDailyAtrFraction`: 0.30
- [ ] Alle Parameter in `application.properties` mit Defaults

### Pipeline-Integration

- [ ] `computeOrRelevanceStatus()` wird aufgerufen nach OR-Level-Erzeugung und TouchRegistry-Befuellung
- [ ] PARKED-Level werden vor `fuseLevelsIntoZones()` entfernt
- [ ] Pipeline-Reihenfolge:
  ```
  1. updateOpeningRanges()           (unveraendert — OR-Level werden IMMER erzeugt)
  2. addOrUpdateStaticLevels()       (unveraendert — OR-Level als SrLevel angelegt)
  3. detectTouches() -> TouchRegistry
     ...
  N. computeOrRelevanceStatus()     (NEU — nach Touch-Erkennung)
     -> PARKED Level entfernen
     ...
  M. fuseLevelsIntoZones()          (nur noch PROTECTED/ACTIVE Level)
  ```

### Erwartetes Ergebnis fuer IREN 23.02

| Zone | Abstand | Status | Aktion |
|------|---------|--------|--------|
| $38.96 OR | $3.04 (7.2%) | PARKED | Entfernt (weit unter jedem Threshold) |
| $39.64 OR | $2.36 (5.6%) | PARKED | Entfernt |
| $39.72 OR | $2.28 (5.4%) | PARKED | Entfernt |
| $40.32 OR | $1.68 (4.0%) | PARKED | Entfernt |
| $40.47 OR | $1.53 (3.6%) | PARKED | Entfernt |

Alle 5 OR-Level werden als PARKED klassifiziert. 5 freie Slots fuer Cluster-Level mit echten Touches.

## Technische Details

**Neue Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/sr/OrRelevanceStatus.java` — Enum
- `odin-brain/src/main/java/de/its/odin/brain/sr/OrRelevanceGate.java` — Hysterese-Logik, Status-Bestimmung, MID-Konditionierung

**Geaenderte Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelEngine.java` — Pipeline-Integration (Aufruf von computeOrRelevanceStatus, PARKED-Entfernung)
- `odin-brain/src/main/java/de/its/odin/brain/config/SrProperties.java` — `OrRelevanceProperties` als verschachtelter Record
- `odin-brain/src/main/resources/application.properties` — Defaults fuer `odin.brain.sr.or-relevance.*`

**Patterns:**
- `OrRelevanceGate` ist ein reines POJO (kein Spring Bean), instanziiert pro Pipeline
- Nutzt `TouchRegistry` (aus ODIN-058) fuer die PROTECTED-Bestimmung via `countValidated()`
- Haelt den vorherigen Status pro OR-Level in einer Map (fuer Hysterese-Entscheidung)
- MID-Generierung ist ein Check in der OR-Level-Erzeugung, nicht ein nachtraeglicher Filter

**Abhaengigkeiten:**
- `TouchRegistry` (ODIN-058) — fuer `countValidated(lower, upper, minQuality)`
- `SrProperties.OrRelevanceProperties` — fuer Konfigurationsparameter
- Bestehende OR-Level-Erzeugung in SrLevelEngine

## Konzept-Referenzen

- `specs/sr-engine-v2/01-concept-or-level-flooding.md` — **gesamtes Dokument**: Problemstellung, Gemini-Kritik, Loesungsansatz, Status-Bestimmung, Hysterese, MID-Korrektur, NMS-Verhalten, Pipeline-Flow
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` — Abschnitt "Status-Bestimmung (dynamisch, pro Bar)": OrRelevanceStatus-Enum und Zustandstabelle
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` — Abschnitt "Hysterese-Gate (Flickering-Schutz)": Asymmetrische Schwellen mit Pseudocode und Zahlenbeispiel
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` — Abschnitt "MID-Level: Konditionale Generierung (korrigiert)": Daily-ATR-Proxy-Formel
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` — Abschnitt "Keine Sonder-Quota": Begruendung warum kein unconfirmedOrMax
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` — Abschnitt "NMS-Verhalten: Kein Sonderschutz": Kein keepAlways fuer OR
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` — Abschnitt "Konfigurationsparameter (SrProperties)": OrRelevanceProperties Record mit Defaults
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` — Abschnitt "Architektur: TouchRegistry (neu)": TouchRegistry-API (`countValidated`)
- `specs/sr-engine-v2/05-concept-output-quality.md` — Abschnitt "Baustein 1: Minimum-Confidence-Threshold": Wie ACTIVE OR-Level natuerlich reguliert werden

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen: neue Klassen in `de.its.odin.brain.sr`
- `docs/backend/guardrails/development-process.md` — Risikoeinstufung HIGH fuer Trading-Logic, ChatGPT-Sparring Pflicht, Gemini 3-Dimensionen-Review Pflicht
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln (R1-R13): kein `var`, keine Magic Numbers, Records fuer DTOs, JavaDoc, Port-Abstraktion
- `T:\codebase\its_odin\CLAUDE.md` — Konfiguration (CSpec): `@ConfigurationProperties` als Record, Namespace `odin.brain.sr.or-relevance.*`

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (`OrRelevanceStatus` als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.sr.or-relevance.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: OR-Level 5% vom Preis entfernt + 0 Touches --> PARKED, nicht in Pipeline
- [ ] Unit-Test: OR-Level 1% vom Preis entfernt + 0 Touches --> ACTIVE, niedrige Confidence
- [ ] Unit-Test: OR-Level 5% entfernt + 2 Touches (Registry) --> PROTECTED, konkurriert normal
- [ ] Unit-Test: Hysterese -- ACTIVE bei Abstand 4.5 ATR --> bleibt ACTIVE (< parkThreshold)
- [ ] Unit-Test: Hysterese -- PARKED bei Abstand 4.5 ATR --> bleibt PARKED (> activateThreshold)
- [ ] Unit-Test: PROTECTED OR + VWAP +-2 Cent --> NMS merged normal (kein keepAlways-Schutz)
- [ ] Unit-Test: Enge OR (< 0.30 x Daily ATR) --> MID wird generiert
- [ ] Unit-Test: Breite OR (> 0.30 x Daily ATR) --> MID wird NICHT generiert
- [ ] Unit-Test: Preis naehert sich geparktem OR --> PARKED-->ACTIVE Transition bei activateThreshold
- [ ] Unit-Test: Initialer Status -- Level innerhalb activateThreshold startet als ACTIVE
- [ ] Unit-Test: Initialer Status -- Level ausserhalb activateThreshold startet als PARKED
- [ ] Unit-Test: Daily-ATR-Proxy korrekt berechnet: max(atr5m * sqrt(78), sessionRange)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer TouchRegistry

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendiger Pipeline-Durchlauf mit realer TouchRegistry und realem OrRelevanceGate -- OR-Level werden korrekt gefiltert
- [ ] Integrationstest: IREN-23.02-aehnliches Setup (5 OR-Level, alle > 3% vom Preis) --> alle PARKED, keines in finaler Ausgabe
- [ ] Integrationstest: Preis-Approach-Szenario -- Preis bewegt sich auf ein PARKED-Level zu --> Transition PARKED-->ACTIVE-->PROTECTED nachvollziehbar
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: OrRelevanceGate-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei ATR = 0? Preis genau auf einer Schwelle? Alle OR-Level PARKED? Nur ein OR-Level (HIGH, kein LOW)? OR-Level mit negativem Abstand (Preis unter OR)?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, Hysterese-Logik-Fehler (falsche Richtung der Vergleiche), Race Conditions beim Status-Speichern, Edge Cases bei ATR nahe 0"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `specs/sr-engine-v2/01-concept-or-level-flooding.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche: OrRelevanceStatus-Werte, Hysterese-Schwellen und Defaults, MID-Konditionierung mit Daily-ATR-Proxy, kein keepAlways, kein Sonder-Quota, Pipeline-Reihenfolge"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Was passiert bei Instrumenten mit sehr breiter Opening Range (> 3%)? Was bei Gap-Up/Gap-Down wo OR weit vom Preis entfernt ist? Interaktion mit Pre-Market-Leveln?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Status-Speicherung pro Level, MID-Check-Zeitpunkt, Initialer-Status-Logik)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **ODIN-058 muss fertig sein.** Diese Story setzt die `TouchRegistry` mit `countValidated()` voraus. Ohne funktionsfaehige Registry kann die PROTECTED-Bestimmung nicht implementiert werden.
- **Hysterese braucht State.** Der vorherige Status jedes OR-Levels muss zwischen Snapshots gespeichert werden. Eine `Map<LevelId, OrRelevanceStatus>` im `OrRelevanceGate` ist der naheliegende Ansatz. Bei einem neuen OR-Level (erster Snapshot) muss der initiale Status aus der Distanz bestimmt werden (ACTIVE wenn innerhalb activateThreshold, sonst PARKED).
- **MID-Generierung aendern, nicht filtern.** Die MID-Konditionierung soll in der OR-Level-Erzeugung greifen — das MID-Level wird gar nicht erst erzeugt wenn die OR-Range zu breit ist. Es soll NICHT nachtraeglich gefiltert werden, da es sonst in der TouchRegistry Spuren hinterlassen koennte.
- **Daily-ATR-Proxy:** Die Formel `atr5m × sqrt(78)` ist eine Volatilitaetsskalierung (nicht linear!). 78 Bars a 5 Minuten = 6.5 Stunden RTH. Die sqrt-Skalierung kommt aus der Volatilitaetstheorie (Random Walk). Der Proxy wird konservativer mit `max(proxy, sessionRange)` — die beobachtete Range kann nicht kleiner als die Schaetzung sein.
- **Kein keepAlways:** Das Konzept ist hier sehr explizit. VWAP behaelt seinen keepAlways-Status, aber OR-Level NICHT. Wenn ein PROTECTED OR-Level und VWAP auf fast dem gleichen Preis liegen, darf NMS sie normal mergen.
- **Interaktion mit Konzept 5:** ACTIVE OR-Level (0 Touches) bekommen durch die Confidence-Berechnung automatisch eine niedrige Confidence. Die Output Policy (ODIN-062) wird sie natuerlich aussortieren wenn es staerkere Level gibt. Das ist gewollt — kein Workaround noetig.
- **Playwright-Test:** Diese Story hat einen Playwright-Test als Akzeptanzkriterium. Der E2E-Test soll verifizieren, dass im Frontend keine OR-Level > 3% vom aktuellen Preis sichtbar sind und die Gesamtzahl der sichtbaren Levels reduziert ist.

### Playwright-Test (PFLICHT)

- [ ] Screenshot des Charts mit S/R-Levels nach einem Backtest-Run mit IREN 23.02
- [ ] Verifikation: Keine OR-Level > 3% vom aktuellen Preis sichtbar im Chart
- [ ] Verifikation: Gesamtzahl der sichtbaren Levels ist reduziert gegenueber Baseline (vorher 12, nachher <= 8)
- [ ] Testdatei: `T:/codebase/its_odin/its-odin-ui/e2e/or-level-relevance.spec.ts`
