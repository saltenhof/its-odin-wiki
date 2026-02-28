# ODIN-030: Session Boundary Lifecycle Integration

**Modul:** odin-core
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-008 (Session Boundaries)

---

## Context

Der LifecycleManager steuert den Tagesablauf aber nutzt die Session-Phasen nicht systematisch. Das Konzept definiert 6 Tages-Phasen mit unterschiedlichen Regeln pro Phase. Die Integration in den Lifecycle muss sicherstellen, dass Pipeline-Zustaende korrekt mit Session-Phasen synchronisiert sind.

## Scope

**In Scope:**
- LifecycleManager reagiert auf SessionPhase-Wechsel
- PRE_MARKET: Pipelines starten, WARMUP beginnt, Instrument-Profiling ausloesen
- RTH_OPENING: WARMUP -> OBSERVING Transition, erste Entries erlaubt
- CORE_TRADING: Normaler Trading-Modus
- POWER_HOUR: Optionaler konservativerer Modus (konfigurierbar)
- RTH_CLOSE_COUNTDOWN: -> FORCED_CLOSE (delegiert an ODIN-027)
- AFTER_HOURS: -> EOD, alle Pipelines gestoppt

**Out of Scope:**
- SessionBoundary-Berechnung selbst (das ist ODIN-008)
- EOD-Close-Sequenz (das ist ODIN-027, wird delegiert)
- Pre-Market-Datenerfassung und Instrument-Profiling-Logik selbst (wird ausgeloest, nicht implementiert)

## Acceptance Criteria

- [ ] LifecycleManager erkennt SessionPhase-Wechsel ueber SessionPhaseResolver
- [ ] PRE_MARKET Start: Pipelines initialisieren, WARMUP starten
- [ ] RTH_OPENING: Pre-Market Instrument Profiling muss abgeschlossen sein
- [ ] POWER_HOUR: Optionale Konfiguration (z.B. konservativere Entry-Schwellen)
- [ ] SessionPhase-Wechsel wird im EventLog dokumentiert
- [ ] Kein Trading in AFTER_HOURS (alle Pipelines gestoppt)

## Technical Details

**Dateien:**
- `odin-core/src/main/java/de/its/odin/core/service/LifecycleManager.java` (Erweiterung)
- `odin-core/src/main/java/de/its/odin/core/config/CoreProperties.java` (Phase-Config)

**Konfiguration:**
- `odin.core.session.power-hour-conservative-mode-enabled` (Default: false)
- `odin.core.session.rth-opening-warmup-minutes` (Default: konfigurierbarer Wert aus ODIN-008)

**Patterns:**
- ALLE Zeitberechnungen ueber `MarketClock`-Port (Sim-Kompatibilitaet)
- SessionPhase-Wechsel-Detection via periodischen Poll oder Event-Callback (je nach ODIN-008 Impl.)
- LifecycleManager ist Singleton — koordiniert alle Pipelines

## Concept References

- `docs/concept/03-strategy-logic.md` -- Abschnitt 1 "Tagesablauf: 6 Phasen"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 8 "Pre-Market-Regeln"
- `docs/concept/00-overview.md` -- Abschnitt 2 "EOD-Flat"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, odin-core Abhaengigkeitsregeln
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints
- `T:\codebase\its_odin\CLAUDE.md` -- "MarketClock verwenden: kein Instant.now() im Trading-Codepfad"

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (`SessionPhase` als ENUM)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.core.session.*` fuer Konfiguration
- [ ] Kein `Instant.now()` im Code — ausschliesslich `MarketClock.now()` verwenden

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: Alle 6 Phasen-Uebergaenge werden korrekt erkannt und behandelt
- [ ] Unit-Tests: PRE_MARKET -> WARMUP-Start fuer alle Pipelines
- [ ] Unit-Tests: RTH_OPENING -> OBSERVING-Transition (nur wenn Profiling abgeschlossen)
- [ ] Unit-Tests: AFTER_HOURS -> alle Pipelines gestoppt, kein Trading moeglich
- [ ] Unit-Tests: EventLog-Events bei jedem Phase-Wechsel
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `MarketClock`, `EventLog`, `SessionPhaseResolver` Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `LifecycleManager` + `SessionPhaseResolver` (real) + `SimClock` — kompletten Tagesablauf von PRE_MARKET bis AFTER_HOURS durchlaufen
- [ ] Integrationstest: Phase-Wechsel triggern korrekte Pipeline-State-Transitionen
- [ ] Integrationstest: POWER_HOUR-Modus aktiviert/deaktiviert Entry-Schwellen korrekt (wenn enabled)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — `LifecycleManager` und Session-Phase-Logik sind In-Memory. Events werden ueber `EventLog`-Port geschrieben (in Tests gemockt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `LifecycleManager` (Session-Phase-Erweiterung), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn RTH_OPENING erkannt wird, aber Profiling noch laeuft? Was bei verspaetem PRE_MARKET-Start (z.B. Systemstart kurz vor RTH)?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions bei Phase-Wechsel-Handling, korrekte Behandlung von verspaetetem Systemstart, Null-Safety bei SessionPhaseResolver"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/03-strategy-logic.md` (Abschnitt 1, 8) + `docs/concept/00-overview.md` (Abschnitt 2) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 6 Phasen korrekt abgebildet sind, ob EOD-Flat-Garantie durch AFTER_HOURS-Handling eingehalten wird"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche Session-Boundary-Szenarien koennten in der Praxis auftreten, die im Konzept nicht behandelt sind? (z.B. Feiertage, vorzeitiger Market-Close, Systemstart nach RTH-Open)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Poll vs. Event-Callback fuer Phase-Detection)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- Alle Zeitberechnungen ueber MarketClock + SessionBoundaries
- Power-Hour-Konfiguration ist optional (kann in V1 ignoriert werden, Default = gleich wie CORE_TRADING)
