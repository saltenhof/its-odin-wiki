# ODIN-007: 1-Minute Monitor Event Detection in Data Pipeline

**Modul:** odin-data
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-005 (MonitorEventType, MonitorEvent)

---

## Context

Die Data Pipeline verarbeitet aktuell 1m-Bars fuer Aggregation zu 3m/5m. Im Drei-Layer-Timeframe-Modell (Stakeholder-Entscheidung 2026-02-26) ist der 1m-Layer der **Event-Detektor** — er laeuft parallel zum Decision-Layer (3m oder 5m, parametrisierbar) und prueft bei jedem 1m-Bar-Close auf extreme Ereignisse. Das Konzept fordert 9 Event-Detektoren auf 1m-Bars, die zwischen den Decision-Bars laufen und LLM-Calls triggern, Alerts emittieren oder den Kill-Switch eskalieren koennen. Dies erfordert eine neue Komponente `MonitorEventDetector` in odin-data.

## Scope

**In Scope:**
- Neue Klasse `MonitorEventDetector` in `de.its.odin.data.service`
- Integration in `DataPipelineService.onBarComplete()` fuer 1m-Bar-Events
- 9 Detektoren: Volume Spike, Price Break Level, Momentum Divergence, Spread Expansion, Stale Quote Warning, Unusual Activity, Range Expansion, VWAP Cross, Exhaustion Signal
- Emission von `MonitorEvent` ueber neuen `MonitorEventListener`-Port
- Konfigurierbare Schwellen pro Detektor via `DataProperties`
- Cooldown-Mechanismus: max 1 Event pro Typ pro konfigurierbarem Zeitfenster (Default 5 Minuten)

**Out of Scope:**
- Vollstaendige Exhaustion-Signal-Implementierung (3-Pillar) -- kommt in ODIN-015 (hier nur einfacher Proxy)
- LLM-Call-Ausloesung selbst (wird von odin-core als Reaktion auf MonitorEvent implementiert)
- Persistierung von MonitorEvents

## Acceptance Criteria

- [ ] `MonitorEventDetector` implementiert alle 9 Detektoren
- [ ] Jeder Detektor hat konfigurierbare Schwellen ueber `DataProperties`
- [ ] Volume Spike: `volume > sma20_volume * volumeSpikeMultiplier` (Default 3.0)
- [ ] Price Break Level: Preis durchbricht ein Key-Level (Intraday High/Low, VWAP)
- [ ] Momentum Divergence: Preis macht neues Hoch, RSI sinkt (oder umgekehrt)
- [ ] Spread Expansion: Spread > spreadAlertThreshold * normalSpread
- [ ] Stale Quote Warning: Kein Tick seit > 30s waehrend RTH
- [ ] VWAP Cross: Preis kreuzt VWAP (oben nach unten oder umgekehrt)
- [ ] Events werden an `MonitorEventListener` emittiert (neuer Port in odin-api)
- [ ] Kein Event-Flooding: Cooldown pro Event-Typ (Default max 1 pro 5 Minuten)

## Technical Details

**Dateien:**
- `odin-data/src/main/java/de/its/odin/data/service/MonitorEventDetector.java`
- `odin-data/src/main/java/de/its/odin/data/config/DataProperties.java` (Erweiterung)

Integration in `DataPipelineService`: Nach jedem 1m-Bar-Close wird `MonitorEventDetector.evaluate(bar, snapshot)` aufgerufen. Events werden an den registrierten Listener weitergeleitet.

## Concept References

- `docs/concept/01-data-pipeline.md` -- Abschnitt 9 "1m Event-Detektor" (ehemals "1-Minute Monitor Events")
- `docs/concept/01-data-pipeline.md` -- Abschnitt 9, Tabelle "9 Event-Typen mit Schwellen und Eskalationsregeln"
- `docs/concept/04-llm-integration.md` -- Abschnitt 4 "LLM-Call-Kadenz: Event-getriggert"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "odin-data: Fachmodul, abhaengig nur von odin-api"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- "MarketClock verwenden: kein Instant.now() im Trading-Codepfad"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-data`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten fuer alle Default-Schwellenwerte
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.data.monitor.{property}` fuer alle Konfigurationsfelder
- [ ] Keine Abhaengigkeit von anderen Fachmodulen -- nur odin-api erlaubt
- [ ] `MarketClock` wird fuer alle Zeitberechnungen verwendet -- kein `Instant.now()`

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer jeden der 9 Detektoren: je mindestens 2 Tests (Trigger-Fall und Nicht-Trigger-Fall)
- [ ] Unit-Test: Cooldown-Logik -- zweites Event desselben Typs innerhalb Cooldown-Fenster wird nicht emittiert
- [ ] Unit-Test: Cooldown-Logik -- Event wird wieder emittiert nachdem Cooldown abgelaufen
- [ ] Unit-Test: Kein Event-Flooding bei schnell aufeinanderfolgenden identischen Bars
- [ ] Unit-Test: Momentum Divergence -- Preis neues Hoch + RSI gesunken = Event
- [ ] Unit-Test: VWAP Cross -- korrekter Richtungswechsel erkannt
- [ ] Testklassen-Namenskonvention: `MonitorEventDetectorTest` (Surefire)
- [ ] Mocks fuer Port-Interfaces (`MonitorEventListener`)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `MonitorEventDetector` eingebettet in `DataPipelineService` -- Volume Spike auf synthetischen Bars loest Event aus
- [ ] Integrationstest: VWAP Cross Event fliegt durch den Stack bis zum registrierten `MonitorEventListener`
- [ ] Integrationstest: Mehrere Detektoren laufen gleichzeitig auf derselben Bar-Sequenz ohne gegenseitige Beeinflussung
- [ ] Testklassen-Namenskonvention: `MonitorEventDetectorIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- `MonitorEventDetector` greift nicht auf die Datenbank zu. Events werden nur emittiert, nicht persistiert.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `MonitorEventDetector.java`, `DataProperties`-Erweiterung, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. erster Bar ohne Historienkontext, Volume = 0, VWAP = Preis exakt, Stale Quote direkt bei RTH-Open, Cooldown-Reset bei neuem Handelstag)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions im Cooldown-State, Null-Safety bei fehlenden Historien-Werten, Ressource-Leaks bei stateful Detektoren"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/01-data-pipeline.md` Abschnitt 8 (inkl. Tabelle) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 9 Detektoren korrekt implementiert sind, die Schwellen der Konzept-Tabelle entsprechen, und das Cooldown-Verhalten dem Konzept entspricht"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Verhalten bei Simulation vs. Live (andere Zeitbasis fuer Cooldown), Initialisierungsphase ohne ausreichend Historien-Bars, Performance bei 9 gleichzeitigen Detektoren"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. ob Cooldown-State pro Detektor oder zentral, wie mit Exhaustion Signal Proxy umgegangen wird)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Die Detektoren muessen stateful sein (Rolling-Referenzwerte fuer SMA, letzte RSI-Werte etc.)
- Cooldown: Einfacher `Map<MonitorEventType, Instant>` mit letztem Emissionszeitpunkt -- MarketClock verwenden, nicht `Instant.now()`
- Stale Quote Detection ueberschneidet sich mit dem bestehenden `StaleQuoteGate` -- aber der Monitor emittiert ein WARNING, das Gate blockiert Daten. Verschiedene Schwellen!
- Exhaustion Signal ist komplex (3-Pillar) -- hier nur ein einfacher Proxy. Die volle Exhaustion Detection kommt in ODIN-015
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
