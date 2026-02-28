# ODIN-009: Pattern Feature Recognition in Data Pipeline

**Modul:** odin-data
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine

---

## Context

Das Konzept definiert 6 Pattern Features (Flush, Reclaim, Coiling, Higher-Low, Breakout-Strength, Pullback-Depth) die von der Data Pipeline berechnet und im MarketSnapshot bereitgestellt werden. Diese sind Inputs fuer die Pattern-FSMs in odin-brain. Aktuell berechnen die Pattern-FSMs ihre Signale selbst aus Roh-Indikatoren -- die dedizierte Feature-Berechnung in odin-data fehlt.

## Scope

**In Scope:**
- Neue Klasse `PatternFeatureCalculator` in `de.its.odin.data.service`
- 6 Pattern Features als Record `PatternFeatures` im API-Modul
- Integration in `MarketSnapshotFactory` -- PatternFeatures werden pro Snapshot berechnet
- Erweiterung von `MarketSnapshot` um `PatternFeatures patternFeatures`

**Out of Scope:**
- Pattern-FSMs in odin-brain (kommen in ODIN-012, ODIN-013) -- hier nur die Feature-Berechnung als Input
- Entry-Signale aus den Features (FSM-Logik, nicht hier)
- Vollstaendige Exhaustion-Signal-Implementierung (kommt in ODIN-015)
- Persistierung von PatternFeatures

## Acceptance Criteria

- [ ] Record `PatternFeatures` mit: `flushDetected` (boolean), `reclaimDetected` (boolean), `coilingScore` (double 0-1), `higherLowConfirmed` (boolean), `breakoutStrength` (double), `pullbackDepth` (double als % von vorherigem Move)
- [ ] Flush-Erkennung: Preis faellt > 2 ATR in < 5 Bars
- [ ] Reclaim-Erkennung: Nach Flush, Preis kehrt ueber Flush-Startlevel zurueck
- [ ] Coiling: Bollinger Bandwidth < Schwelle fuer > N Bars
- [ ] Higher-Low: Aktueller Swing-Low > vorheriger Swing-Low
- [ ] Breakout-Strength: Volume * Range am Breakout-Bar relativ zum Durchschnitt
- [ ] Pullback-Depth: Aktueller Rueckzug als Prozent des vorherigen Up-Move (Fibonacci-artig)

## Technical Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/dto/PatternFeatures.java` (neues Record)
- `odin-data/src/main/java/de/its/odin/data/service/PatternFeatureCalculator.java` (neue Klasse)
- `odin-data/src/main/java/de/its/odin/data/service/MarketSnapshotFactory.java` (Erweiterung)
- `odin-api/src/main/java/de/its/odin/api/dto/MarketSnapshot.java` (Erweiterung)

## Concept References

- `docs/concept/01-data-pipeline.md` -- Abschnitt 7 "Pattern Feature Recognition"
- `docs/concept/01-data-pipeline.md` -- Abschnitt 7, Tabelle "6 Pattern Features"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 6 "4 Setups (A-D)"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "odin-data: Fachmodul, abhaengig nur von odin-api"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-data`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten fuer alle Schwellenwerte (z.B. `FLUSH_ATR_MULTIPLIER = 2.0`, `FLUSH_MAX_BARS = 5`)
- [ ] Records fuer DTOs (`PatternFeatures` als Record)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen -- Methoden-JavaDoc beschreibt den Algorithmus und die Eingabeanforderungen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.data.pattern.{property}` fuer alle Konfigurationsfelder
- [ ] Keine Abhaengigkeit von anderen Fachmodulen -- nur odin-api erlaubt

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer jedes der 6 Features mit synthetischen Bar-Daten
- [ ] Unit-Test Flush: Bar-Sequenz die > 2 ATR in < 5 Bars faellt -> `flushDetected=true`
- [ ] Unit-Test Flush: Bar-Sequenz die nur 1 ATR faellt -> `flushDetected=false`
- [ ] Unit-Test Reclaim: Nach Flush, Preis ueber Startlevel -> `reclaimDetected=true`
- [ ] Unit-Test Reclaim: Nach Flush, Preis noch unter Startlevel -> `reclaimDetected=false`
- [ ] Unit-Test Coiling: Konstante enge Bollinger-Bandwidth ueber N Bars -> `coilingScore` > 0
- [ ] Unit-Test Higher-Low: Synthetische Swing-Lows aufsteigend -> `higherLowConfirmed=true`
- [ ] Unit-Test Breakout-Strength: Erster Bar ohne Historien-Kontext (Initialzustand)
- [ ] Testklassen-Namenskonvention: `PatternFeatureCalculatorTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `PatternFeatureCalculator` + `MarketSnapshotFactory` -- vollstaendiger Snapshot enthaelt korrektes `PatternFeatures`-Objekt
- [ ] Integrationstest: Flush gefolgt von Reclaim -- beide Features korrekt erkannt in aufeinanderfolgenden Snapshots
- [ ] Integrationstest: `coilingScore` steigt kontinuierlich bei anhaltend enger Bollinger Bandwidth
- [ ] Testklassen-Namenskonvention: `PatternFeatureIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- `PatternFeatureCalculator` greift nicht auf die Datenbank zu. Berechnungen basieren ausschliesslich auf In-Memory-Bar-Daten.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `PatternFeatureCalculator.java`, `PatternFeatures.java`, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Flush direkt bei Marktopen ohne Historienkontext, Reclaim ohne vorangehenden Flush, Coiling mit nur 1 Bar, Breakout-Strength bei Volume=0, negative Pullback-Depth bei neuem Hoch)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs bei der stateful Historien-Verwaltung (Swing-Highs/Lows), Off-by-one-Fehler bei Bar-Zaehlungen, Division-by-zero bei Breakout-Strength/Pullback-Depth, Null-Safety bei unzureichender Bar-Historien"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/01-data-pipeline.md` Abschnitt 7 (inkl. Tabelle) + `docs/concept/03-strategy-logic.md` Abschnitt 6 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 6 Pattern Features korrekt implementiert sind und die Algorithmen der Konzept-Tabelle entsprechen. Sind die Features als reine Inputs (nicht Entscheidungen) implementiert?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. wie viel Historien-Bars benoetigt werden bevor Features reliable sind (warm-up), Performance bei sehr vielen gecachten Swing-Punkten, Verhalten bei Gaps zwischen Bars"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. wie Swing-High/Low-Detection implementiert wird, wie viel Bar-Historien gespeichert wird, ob Reclaim Flush-State benoetigt)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- PatternFeatureCalculator muss stateful sein (braucht Historie fuer Swing-Highs/Lows)
- Die Features sind INPUTS fuer die Pattern-FSMs in odin-brain. Die FSMs (ODIN-012, ODIN-013) entscheiden, das Feature liefert nur Rohdaten
- Flush/Reclaim-Detection hier ist einfacher als die FSM-Version in odin-brain -- hier nur boolean "wurde erkannt", dort vollstaendige Zustandsmaschine mit Entry-Signalen
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
