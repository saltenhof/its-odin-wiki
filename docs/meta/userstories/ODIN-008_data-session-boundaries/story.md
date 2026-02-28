# ODIN-008: Session Boundary Detection und RTH-Erkennung

**Modul:** odin-data
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine

---

## Context

Das Konzept definiert klare Tages-Phasen (Pre-Market, RTH-Open, Core-Trading, RTH-Close-Countdown, After-Hours) mit unterschiedlichen Regeln. Die aktuelle `SessionBoundaries`-Port-Definition existiert aber die Implementierung nutzt sie nicht durchgaengig. Die Data Pipeline muss Session-Phasen erkennen und als Kontext an die Decision Loop weiterreichen.

## Scope

**In Scope:**
- Implementierung/Erweiterung von `SessionBoundaries` Port-Nutzung in `DataPipelineService`
- Neues Enum `SessionPhase` (PRE_MARKET, RTH_OPENING, CORE_TRADING, POWER_HOUR, RTH_CLOSE_COUNTDOWN, AFTER_HOURS)
- Session-Phase als Feld in `MarketSnapshot`
- Konfigurierbare Zeitfenster pro Exchange (NYSE Default: Pre 04:00-09:30, RTH 09:30-16:00 ET)
- Forced-Close-Countdown-Erkennung (letzte 15 Minuten RTH)

**Out of Scope:**
- EU-Markt-Support (PHASE2, ODIN-046)
- Session-Phase-basierte Entry/Exit-Regeln (werden in odin-brain-Stories genutzt)
- Persistierung von Session-Phasen

## Acceptance Criteria

- [ ] Enum `SessionPhase` in odin-api mit 6 Werten: PRE_MARKET, RTH_OPENING, CORE_TRADING, POWER_HOUR, RTH_CLOSE_COUNTDOWN, AFTER_HOURS
- [ ] `MarketSnapshot` enthaelt `SessionPhase sessionPhase`
- [ ] `DataPipelineService` bestimmt SessionPhase basierend auf MarketClock + SessionBoundaries
- [ ] RTH_CLOSE_COUNTDOWN wird 15 Minuten vor RTH-Close erkannt
- [ ] POWER_HOUR: letzte 60 Minuten RTH (15:00-16:00 ET)
- [ ] Pre-Market-Bars werden korrekt als PRE_MARKET markiert
- [ ] Konfigurierbar pro Exchange in `DataProperties`

## Technical Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/model/SessionPhase.java` (neues Enum)
- `odin-api/src/main/java/de/its/odin/api/dto/MarketSnapshot.java` (Erweiterung)
- `odin-data/src/main/java/de/its/odin/data/service/SessionPhaseResolver.java` (neue Klasse)
- `odin-data/src/main/java/de/its/odin/data/service/MarketSnapshotFactory.java` (Erweiterung)

## Concept References

- `docs/concept/03-strategy-logic.md` -- Abschnitt 1 "Tagesablauf: 6 Phasen"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 1, Tabelle "6 Tages-Phasen mit Zeitfenstern"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 8 "Pre-Market-Regeln"
- `docs/concept/00-overview.md` -- Abschnitt 2 "Systemparameter: US-Maerkte, NYSE/NASDAQ"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "MarketClock verwenden: kein Instant.now()"
- `docs/backend/guardrails/module-structure.md` -- "odin-data: Fachmodul, abhaengig nur von odin-api"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- "MarketClock verwenden: kein Instant.now() im Trading-Codepfad"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-data`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten fuer alle Zeitgrenzen (z.B. `CLOSE_COUNTDOWN_MINUTES = 15`)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (`SessionPhase` als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen -- JavaDoc pro Enum-Wert mit Zeitfenster-Beschreibung
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.data.session.{property}` fuer alle Konfigurationsfelder
- [ ] Keine Abhaengigkeit von anderen Fachmodulen -- nur odin-api erlaubt
- [ ] Alle Zeitberechnungen ueber `MarketClock` -- kein `Instant.now()`

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Jede SessionPhase mit konkreten Beispiel-Zeitpunkten (ET-Zeitzone)
- [ ] Unit-Test: RTH_CLOSE_COUNTDOWN bei 15:45 ET (NYSE) -> true
- [ ] Unit-Test: RTH_CLOSE_COUNTDOWN bei 15:44 ET (NYSE) -> false (noch POWER_HOUR)
- [ ] Unit-Test: POWER_HOUR bei 15:00 ET -> true
- [ ] Unit-Test: Grenzwert RTH-Open -- 09:30:00 ET genau -> RTH_OPENING, 09:29:59 ET -> PRE_MARKET
- [ ] Unit-Test: AFTER_HOURS nach 16:00 ET
- [ ] Testklassen-Namenskonvention: `SessionPhaseResolverTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `SessionPhaseResolver` + `MarketSnapshotFactory` -- Snapshot enthaelt korrekte `sessionPhase` fuer verschiedene Zeitpunkte
- [ ] Integrationstest: `DataPipelineService` setzt `sessionPhase` im generierten `MarketSnapshot` korrekt
- [ ] Integrationstest: Simulation-Kompatibilitaet -- `SimClock` mit gesetzter Zeit liefert korrekte SessionPhase (keine Abhaengigkeit von Systemzeit)
- [ ] Testklassen-Namenskonvention: `SessionBoundaryIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- `SessionPhaseResolver` und `MarketSnapshotFactory` greifen nicht auf die Datenbank zu.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `SessionPhaseResolver.java`, `SessionPhase`-Enum, `MarketSnapshotFactory`-Erweiterung, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Daylight-Saving-Time-Uebergang, Mitternacht um Jahreswechsel, feiertagsbedingte Marktschliessungen, Genauigkeit bei Millisekunden-Grenzwerten)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs bei Zeitzonenberechnung, Off-by-one-Fehler bei Phasengrenzen, Null-Safety, DST-Probleme, korrekte Verwendung von MarketClock statt Instant.now()"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/03-strategy-logic.md` Abschnitte 1 und 8 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 6 Session-Phasen korrekt abgebildet sind, Zeitfenster dem Konzept entsprechen, und RTH_CLOSE_COUNTDOWN sowie POWER_HOUR korrekt berechnet werden"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Verhalten bei fruehzeitiger Marktschliessungen (Half Days), Unterschied NYSE vs. NASDAQ Handelszeiten, Early-Close-Erkennung"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Zeitzonenbehandlung, ob RTH_OPENING als eigene Phase oder Untermenge von CORE_TRADING)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Alle Zeitberechnungen MUESSEN ueber MarketClock laufen (Sim-Kompatibilitaet!)
- NYSE: RTH 09:30-16:00 ET, Pre-Market ab 04:00 ET
- SessionBoundaries-Port existiert bereits in odin-api -- nutzen, nicht neu erfinden
- V1 = nur US-Maerkte. EU-Support ist PHASE2 (ODIN-046)
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
